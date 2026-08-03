# Shared Webhook Architecture & Async Reconciliation (GitHub & GitLab)

This guide documents the shared repository webhook architecture (v2) implemented in Terrakube for GitHub and GitLab VCS integrations.

## 1. Overview & Problem Solved
In Terrakube v1, each workspace registered its own webhook directly with GitHub or GitLab. When multiple workspaces were created for the same repository URL, this led to duplicate webhook registrations, VCS API rate limiting, and redundant event handling.

In Terrakube v2 (shared webhooks):
- A single `RepoWebhook` entity is registered per normalized repository URL.
- All workspaces referencing that repo URL (with `isMigratedV2 = true`) share this webhook.
- Webhook management and event fanout are decoupled from Elide transaction boundaries using Quartz asynchronous job scheduling.

---

## 2. Key Components & Implementation Patterns

### A. Elide Lifecycle Hook (`WebhookManageHook.java`)
Location: `io.terrakube.api.rs.hooks.webhook.WebhookManageHook`

- Implements `com.yahoo.elide.core.lifecycle.LifeCycleHook<Webhook>`.
- Triggered on `Webhook` entity `CREATE`, `UPDATE`, and `DELETE`.

```java
switch (operation) {
    case CREATE:
    case UPDATE:
        switch (phase) {
            case PRECOMMIT:
                // Delete legacy v1 per-workspace hook inline if migrating to v2
                if (isMigratedV2Shared(elideEntity)) {
                    if (elideEntity.getRemoteHookId() != null) {
                        deleteWorkspaceHook(elideEntity);
                        elideEntity.setRemoteHookId(null);
                    }
                }
                break;
            case POSTCOMMIT:
                // Schedule async reconciliation after DB transaction commits
                if (isSharedWebhookProvider(elideEntity)) {
                    scheduleSync(elideEntity);
                }
                break;
        }
        break;
    case DELETE:
        switch (phase) {
            case POSTCOMMIT:
                if (isMigratedV2Shared(elideEntity)) {
                    scheduleSync(elideEntity);
                } else {
                    webhookService.deleteWorkspaceWebhook(elideEntity);
                }
                break;
        }
        break;
}
```

> **Key Rule**: Always schedule asynchronous background reconciliation during `POSTCOMMIT`. Scheduling during `PRECOMMIT` would cause background job queries to execute against uncommitted database state.

---

### B. Quartz Coalescing Scheduler (`RepoWebhookSyncScheduler.java`)
Location: `io.terrakube.api.plugin.scheduler.webhook.RepoWebhookSyncScheduler`

- Coalesces rapid/concurrent triggers for the same repository URL into a single background job run.
- Keys jobs deterministically using `SHA-256(normalizedRepositoryUrl)` under job group `"repo-webhook-sync"`.

```java
static JobKey jobKeyFor(String normalizedRepositoryUrl) {
    return new JobKey(sha256Hex(normalizedRepositoryUrl), JOB_GROUP);
}

public void scheduleSync(String normalizedRepositoryUrl, String workspaceId) {
    JobKey jobKey = jobKeyFor(normalizedRepositoryUrl);
    JobDataMap jobDataMap = new JobDataMap();
    jobDataMap.put(DATA_KEY_REPOSITORY_URL, normalizedRepositoryUrl);
    jobDataMap.put(DATA_KEY_WORKSPACE_ID, workspaceId);

    JobDetail jobDetail = JobBuilder.newJob(RepoWebhookSyncJob.class)
            .withIdentity(jobKey)
            .usingJobData(jobDataMap)
            .storeDurably(false)
            .build();

    Trigger trigger = TriggerBuilder.newTrigger()
            .forJob(jobDetail)
            .withIdentity(jobKey.getName(), jobKey.getGroup())
            .startNow()
            .build();

    try {
        scheduler.scheduleJob(jobDetail, trigger);
    } catch (ObjectAlreadyExistsException _) {
        // Coalesced: another job for this exact URL is already scheduled
    } catch (JobPersistenceException e) {
        // Coalesced: lost a race on job store insert
    }
}
```

---

### C. Async Reconciliation Job (`RepoWebhookSyncJob.java`)
Location: `io.terrakube.api.plugin.scheduler.webhook.RepoWebhookSyncJob`

- Annotated `@DisallowConcurrentExecution` to prevent overlapping executions per job key.
- Runs inside `@Transactional(rollbackFor = Exception.class)`.

```java
@Component
@DisallowConcurrentExecution
public class RepoWebhookSyncJob implements Job {
    @Override
    @Transactional(rollbackFor = Exception.class)
    public void execute(JobExecutionContext context) throws JobExecutionException {
        String repositoryUrl = context.getJobDetail().getJobDataMap().getString(DATA_KEY_REPOSITORY_URL);
        List<Workspace> workspaces = workspaceRepository.findByNormalizedSourceWithMigratedWebhook(repositoryUrl);

        if (workspaces.isEmpty()) {
            repoWebhookRepository.findByRepositoryUrl(repositoryUrl)
                    .ifPresentOrElse(repoWebhookService::cleanupIfOrphan, () -> {});
            return;
        }

        Workspace seedWorkspace = findSeedWorkspace(workspaces, workspaceId);
        RepoWebhook repoWebhook = repoWebhookService.getOrCreateRepoWebhook(seedWorkspace);
        repoWebhookService.createOrUpdateSharedWebhook(repoWebhook);
    }
}
```

---

### D. Service Layer & DB Race Condition Handling (`RepoWebhookService.java`)
Location: `io.terrakube.api.plugin.vcs.RepoWebhookService`

- `getOrCreateRepoWebhook()` normalizes repository URLs via `RepoUrlNormalizer.normalize()`.
- Uses `saveAndFlush()` to catch `DataIntegrityViolationException` on `repository_url` unique constraint and recovers by re-querying:

```java
public RepoWebhook getOrCreateRepoWebhook(Workspace workspace) {
    String normalizedUrl = RepoUrlNormalizer.normalize(workspace.getSource());
    Optional<RepoWebhook> existing = repoWebhookRepository.findByRepositoryUrl(normalizedUrl);
    if (existing.isPresent()) return existing.get();

    RepoWebhook newWebhook = new RepoWebhook();
    newWebhook.setRepositoryUrl(normalizedUrl);
    newWebhook.setWebhookSecret(UUID.randomUUID().toString());
    newWebhook.setVcs(workspace.getVcs());

    try {
        return repoWebhookRepository.saveAndFlush(newWebhook);
    } catch (DataIntegrityViolationException e) {
        return repoWebhookRepository.findByRepositoryUrl(normalizedUrl)
                .orElseThrow(() -> new IllegalStateException("Failed to create or find RepoWebhook"));
    }
}
```

---

## 3. Unit Testing Guidelines (`RepoWebhookServiceTest.java`)
Location: `terrakube/api/src/test/java/io/terrakube/api/plugin/vcs/RepoWebhookServiceTest.java`

When writing test cases for shared VCS webhooks:
1. **GetOrCreateRepoWebhook**: Test existing return, new creation, URL normalization, and race-condition recovery (`DataIntegrityViolationException`).
2. **CreateOrUpdateSharedWebhook**: Verify aggregation of `WebhookEventType` sets (`PUSH`, `PULL_REQUEST`, `RELEASE`) across multiple workspaces sharing the repo URL.
3. **CleanupIfOrphan**: Test deletion of both VCS remote webhook and `RepoWebhook` DB entity when 0 migrated workspaces remain.
4. **ProcessV2Webhook**: Test HMAC signature verification failure (`SecurityException`), ping events, PR file change matching, release tag regex matching, and job fanout to multiple matching workspaces.
