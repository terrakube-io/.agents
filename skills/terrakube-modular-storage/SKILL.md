---
name: terrakube-modular-storage
description: Architecture, conventions, and implementation guide for modular cloud storage starters (AWS S3, Azure Blob, GCP Cloud Storage, Local) and container image splitting in Terrakube.
---

# Terrakube Modular Storage Architecture & Container Image Splitting

## Overview
Terrakube manages object storage (Terraform state, modules, providers, plans, workspaces) through a decoupled Spring Boot 3 multi-module architecture located in `terrakube/storage`. This architecture allows components (`registry`, `api`, `executor`) to build cloud-specific container variants (`-Paws`, `-Pazure`, `-Pgcp`) or an all-in-one backwards-compatible container (`-Pall` / default).

---

## Module Layout (`terrakube/storage`)

```
terrakube/storage/
├── pom.xml                     <-- Parent aggregator & Cloud BOM manager
├── storage-core/               <-- StorageService, GitService, ModuleVersionDownload, StorageProperties
├── storage-aws/                <-- AWS S3 StorageService implementation + AutoConfiguration
├── storage-azure/              <-- Azure Blob StorageService implementation + AutoConfiguration
├── storage-gcp/                <-- GCP Cloud Storage implementation + AutoConfiguration
└── storage-local/              <-- Local disk StorageService implementation + AutoConfiguration
```

### Dependency Rules:
1. `storage-core` contains only shared interfaces and models (`StorageService`, `GitService`, `StorageProperties`, `StorageType`). It **MUST NEVER** depend on cloud SDKs (AWS SDK, Azure SDK, Google Cloud Storage, gRPC).
2. `storage-aws`, `storage-azure`, `storage-gcp`, and `storage-local` depend on `storage-core` and import their respective cloud BOMs / SDKs.

---

## Spring Boot 3 Auto-Configuration Pattern

Every storage starter (`storage-aws`, `storage-azure`, `storage-gcp`, `storage-local`) registers its configuration class using the Spring Boot 3 contract:

1. **Imports File**: `src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` containing the fully-qualified class name.
2. **Conditional Annotations**:
```java
@AutoConfiguration
@EnableConfigurationProperties({AwsStorageServiceProperties.class, StorageProperties.class})
@ConditionalOnClass(S3Client.class)
@ConditionalOnProperty(prefix = "io.terrakube.registry.plugin.storage", name = "type", havingValue = "AWS")
public class AwsStorageAutoConfiguration {

    @Bean
    public StorageService awsStorageService(AwsStorageServiceProperties properties) {
        return new AwsStorageServiceImpl(properties);
    }
}
```

---

## Decoupled Cloud SDK Usage via Reflection

When a consuming service (e.g. `GitServiceImpl`) requires an optional cloud credential mechanism (like Azure Identity `DefaultAzureCredentialBuilder`), **never** import cloud classes directly if the service can be built without that provider.

Use dynamic runtime reflection:
```java
private static String getAzureDefaultToken(String audience) {
    try {
        Class<?> builderClass = Class.forName("com.azure.identity.DefaultAzureCredentialBuilder");
        Object builder = builderClass.getDeclaredConstructor().newInstance();
        Object credential = builderClass.getMethod("build").invoke(builder);
        // Invoke getBearerToken or getToken
        ...
        return token.getToken();
    } catch (ClassNotFoundException e) {
        log.debug("Azure identity SDK not available on classpath: {}", e.getMessage());
        return null;
    } catch (Exception e) {
        log.error("Failed to retrieve Azure default credential token", e);
        return null;
    }
}
```

---

## Consuming Service Maven Profile Pattern

Any Terrakube component adopting modular storage must define the following profile layout in its `pom.xml`:

```xml
<dependencies>
    <dependency>
        <groupId>io.terrakube.storage</groupId>
        <artifactId>storage-core</artifactId>
    </dependency>
    <dependency>
        <groupId>io.terrakube.storage</groupId>
        <artifactId>storage-local</artifactId>
    </dependency>
</dependencies>

<profiles>
    <profile>
        <id>all</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <dependencies>
            <dependency>
                <groupId>io.terrakube.storage</groupId>
                <artifactId>storage-aws</artifactId>
            </dependency>
            <dependency>
                <groupId>io.terrakube.storage</groupId>
                <artifactId>storage-azure</artifactId>
            </dependency>
            <dependency>
                <groupId>io.terrakube.storage</groupId>
                <artifactId>storage-gcp</artifactId>
            </dependency>
        </dependencies>
    </profile>
    <profile>
        <id>aws</id>
        <dependencies>
            <dependency>
                <groupId>io.terrakube.storage</groupId>
                <artifactId>storage-aws</artifactId>
            </dependency>
        </dependencies>
    </profile>
    <profile>
        <id>azure</id>
        <dependencies>
            <dependency>
                <groupId>io.terrakube.storage</groupId>
                <artifactId>storage-azure</artifactId>
            </dependency>
        </dependencies>
    </profile>
    <profile>
        <id>gcp</id>
        <dependencies>
            <dependency>
                <groupId>io.terrakube.storage</groupId>
                <artifactId>storage-gcp</artifactId>
            </dependency>
        </dependencies>
    </profile>
</profiles>
```

---

## Building and Testing Commands

Always use `-am` (`--also-make`) so Maven resolves upstream `storage-*` modules in the reactor:

```bash
# Test all storage profiles
mvn test -pl registry -am -Paws -Dspring-boot.build-image.skip=true
mvn test -pl registry -am -Pazure -Dspring-boot.build-image.skip=true
mvn test -pl registry -am -Pgcp -Dspring-boot.build-image.skip=true

# Package split container images
mvn clean package -pl registry -am -Paws -DskipTests
mvn clean package -pl registry -am -Pazure -DskipTests
mvn clean package -pl registry -am -Pgcp -DskipTests
mvn clean package -pl registry -am -Pall -DskipTests
```

---

## Image Tagging Strategy

* `azbuilder/<component>:<version>-aws` (Built with `-Paws`)
* `azbuilder/<component>:<version>-azure` (Built with `-Pazure`)
* `azbuilder/<component>:<version>-gcp` (Built with `-Pgcp`)
* `azbuilder/<component>:<version>` and `:latest` (Built with `-Pall` for full backwards compatibility)
