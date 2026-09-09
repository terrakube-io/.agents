---
name: terrakube-devcontainer
description: Connecting to PostgreSQL, inspecting Redis and MinIO, and debugging database and runtime data issues for terrakube-api.platform.local in the local devcontainer environment.
---

# Terrakube Devcontainer & Local Database Debugging Guide

This skill documents how to connect to the PostgreSQL database, inspect service state, and troubleshoot data issues in `terrakube-api.platform.local` within the local Docker Compose / Devcontainer environment (`terrakube/.devcontainer`).

---

## 1. Local Environment Architecture & Network Topology

The local development stack runs in Docker under the `terrakube-network` bridge network:

| Container / Service | Host Port | In-Network Hostname / Port | Credentials / Notes |
| :--- | :--- | :--- | :--- |
| **`postgresql-service`** | `5432` | `postgresql-service:5432` | User: `terrakube`, DB: `terrakubedb`, Pass: `terrakubepassword` |
| **`terrakube-dev`** | — | `terrakube-dev` | Devcontainer workspace (`/workspaces/terrakube`) |
| **`terrakube-redis`** (Valkey) | Internal | `terrakube-redis:6379` | Auth: `password123456` |
| **`terrakube-minio`** | `9000`, `9001` | `terrakube-minio:9000` | User: `minioadmin`, Pass: `minioadmin`, Bucket: `sample` |
| **`terrakube-traefik`** | `80`, `443` | `terrakube-traefik` | Ingress for `terrakube-api.platform.local`, `terrakube.platform.local` |
| **`dex-service`** | `5556` | `terrakube-dex.platform.local` | OIDC issuer |

---

## 2. PostgreSQL Connection Methods

### Method A: Direct Execution via Docker (Fastest & Zero Setup)
Run interactive `psql` or single-command queries directly on the PostgreSQL container:

**Interactive Shell:**
```bash
docker exec -it postgresql-service psql -U terrakube -d terrakubedb
```

**One-liner Query:**
```bash
docker exec postgresql-service psql -U terrakube -d terrakubedb -c "SELECT id, status, comments, created_date FROM job ORDER BY id DESC LIMIT 5;"
```

---

### Method B: From the Host Machine (via Exposed Port 5432)
The PostgreSQL port is mapped to `0.0.0.0:5432` on the host:

**Using `psql`:**
```bash
PGPASSWORD=terrakubepassword psql -h localhost -p 5432 -U terrakube -d terrakubedb
```

**Connection String (for DBeaver, pgAdmin, IntelliJ, VS Code extensions):**
```
postgresql://terrakube:terrakubepassword@localhost:5432/terrakubedb
```

---

### Method C: From Inside the `terrakube-dev` Devcontainer
Inside the `terrakube-dev` container, the host is `postgresql-service`:

1. **Install `psql` client (if not already installed):**
   ```bash
   sudo apt-get update && sudo apt-get install -y postgresql-client
   ```
2. **Connect via container DNS:**
   ```bash
   PGPASSWORD=terrakubepassword psql -h postgresql-service -p 5432 -U terrakube -d terrakubedb
   ```

---

## 3. Common SQL Diagnostic Queries for `terrakube-api`

### 1. Investigating Stuck or Pending Jobs
Check recent jobs, flow types, and status:
```sql
SELECT id, status, comments, plan_changes, created_by, created_date, updated_date
FROM job
ORDER BY id DESC
LIMIT 10;
```

Check why a job is blocked (inspect active steps):
```sql
SELECT id, step_number, status, name, executor
FROM step
WHERE job_id = <JOB_ID>
ORDER BY step_number ASC;
```

### 2. Workspace Status & Execution Locks
Check if a workspace is locked or what its latest job status is:
```sql
SELECT id, name, last_job_status, locked, deleted
FROM workspace
WHERE name = '<WORKSPACE_NAME>';
```

Unlock a stuck workspace:
```sql
UPDATE workspace SET locked = false WHERE name = '<WORKSPACE_NAME>';
```

### 3. Manual Job Remediation (Cancelling Stalled / Stuck Runs)
If a pending run is blocking the FIFO queue or workspace:
```sql
-- Cancel the job
UPDATE job SET status = 'cancelled' WHERE id = <JOB_ID>;

-- Cancel any lingering steps
UPDATE step SET status = 'cancelled' WHERE job_id = <JOB_ID> AND status IN ('pending', 'running', 'queue');
```

### 4. Policy Evaluations & Violations
Check policy scan results for a job:
```sql
SELECT id, job_id, status, total_policies, passed_policies, failed_policies, violations_url, created_date
FROM policy_evaluation
ORDER BY created_date DESC
LIMIT 5;
```

Check policy overrides:
```sql
SELECT id, job_id, policy_id, approved_by, comment, created_date
FROM policy_override
ORDER BY created_date DESC;
```

### 5. Outbox Notifications (Webhook / Slack Deliveries)
Inspect pending or failed notifications:
```sql
SELECT id, destination_type, status, retry_count, last_error, created_date
FROM notification_outbox
ORDER BY created_date DESC
LIMIT 10;
```

---

## 4. Resetting or Refreshing Database State

To completely reset the local database (wipes all tables and allows Liquibase to re-run from scratch):
```bash
docker compose -f terrakube/.devcontainer/docker-compose.yml stop postgresql-service
docker compose -f terrakube/.devcontainer/docker-compose.yml rm -f -v postgresql-service
docker volume rm devcontainer_postgresql_data
docker compose -f terrakube/.devcontainer/docker-compose.yml up -d postgresql-service
```
