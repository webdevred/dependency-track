# Dependency-Track v5 — Agent Context

## Build environment

Requires **JDK 25** and **Maven 4**. The Temurin distribution is recommended (see `DEVELOPING.md`), but the user prefers installing from the Linux distro repo (`sudo apt install openjdk-25-jdk`) when available.

`protobuf-maven-plugin:5.1.4` (in `dex/dex-api`) uses the Maven 4 API and crashes on Maven 3.9.x with `NullInjectedIntoNonNullable` on `ProtocResolver`. Maven 3.9 is what most distro package managers install — it will not work. Get Maven 4:

```bash
curl -sL "https://downloads.apache.org/maven/maven-4/4.0.0-rc-5/binaries/apache-maven-4.0.0-rc-5-bin.tar.gz" \
  | tar xzf - -C /opt/
export MVN=/opt/apache-maven-4.0.0-rc-5/bin/mvn
```

The `make` targets hardcode the system `mvn`, so run Maven directly:

```bash
# Run a single test
$MVN -B -q -Dsurefire.useFile=false test \
  -Dmaven.build.cache.enabled=false -Dcheckstyle.skip -Dcyclonedx.skip \
  -pl apiserver -am -Dtest="SomeTest"

# Build JAR and Docker image
$MVN -B -q -Pquick package -pl apiserver -am
docker build -f apiserver/src/main/docker/Dockerfile -t dependencytrack/apiserver:local apiserver/
```

The Dockerfile uses `--chmod` which requires BuildKit. Install `docker-buildx-plugin` (`sudo apt install docker-buildx-plugin`) if the build fails with a syntax error on that line.

## Repository structure (v5)

- `apiserver/` — the main application module, where almost all feature work happens
- `alpine/` — internal shared libraries (model, infra, server)
- `proto/` — Protobuf definitions for internal messaging
- `dex/` — OIDC provider (dex)
- `migration/` — database migration scripts

## Persistence

v5 uses **JDBI with named-column mapping** for new persistence code. Query results are mapped by column name, not positional index, so adding a column to a SELECT list does not affect other fields. The old JDO/DataNucleus layer (`@PersistenceCapable` models) is legacy — avoid touching it.

## Testing patterns

- Base class for persistence tests: `PersistenceCapableTest`
- HTTP mocking: WireMock via `@WireMockTest` — `WireMockRuntimeInfo` is injected as a **method parameter**, not a field
- Secret injection in tests: `TestSecretManager(Map.of("secretName", "secretValue"))` — the config property holds the secret *name*, the manager resolves it to the value
- Task invocation in tests: `new SomeTask(HttpClient.newHttpClient(), new TestSecretManager(...)).run()` — no no-arg constructors, no `inform()` method (that was v4)

## Local end-to-end demo (DT + DefectDojo)

The compose setup lives in `dev/`. The base file is `dev/compose.yaml` (not `docker-compose.yml`).

```bash
# Start full stack with DefectDojo and a locally built apiserver image:
docker compose \
  -f dev/compose.yaml \
  -f dev/docker-compose.defectdojo.yml \
  -f dev/docker-compose.local.yml \
  up -d
```

- **DT UI:** http://localhost:8080
- **DefectDojo UI:** http://localhost:8082 — credentials `admin / admin`
- DefectDojo takes ~2 min on first run before the UI is usable

**DT first-run password change:** On a fresh database the admin account requires a forced password change before the API accepts logins. Do it via API:

```bash
curl -X POST http://localhost:8080/api/v1/user/forceChangePassword \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'username=admin&password=admin&newPassword=<new>&confirmPassword=<new>'
```

**Configuring the DT→DefectDojo integration via API:**

The `defectdojo.apiKey` config property holds the *name* of a secret, not the key itself. The actual key is stored separately via the v2 secrets API and resolved at runtime by `SecretManager`.

```bash
# 1. Store the DD token as a named secret
curl -X POST http://localhost:8080/api/v2/secrets \
  -H "Authorization: Bearer $DT_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"name":"defectdojo-api-key","value":"<dd-token>"}'

# 2. Point the config property at the secret name (type must be STRING, not SECRET)
curl -X POST http://localhost:8080/api/v1/configProperty \
  -H "Authorization: Bearer $DT_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"groupName":"integrations","propertyName":"defectdojo.apiKey","propertyValue":"defectdojo-api-key","propertyType":"STRING"}'

# 3. Set URL and enable
curl -X POST http://localhost:8080/api/v1/configProperty \
  -H "Authorization: Bearer $DT_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"groupName":"integrations","propertyName":"defectdojo.url","propertyValue":"http://defectdojo-nginx:8080","propertyType":"URL"}'

curl -X POST http://localhost:8080/api/v1/configProperty \
  -H "Authorization: Bearer $DT_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"groupName":"integrations","propertyName":"defectdojo.enabled","propertyValue":"true","propertyType":"BOOLEAN"}'
```

**Triggering uploads without waiting for 2 AM:** The DefectDojo upload task runs daily at 02:00 UTC by default. Override in `dev/docker-compose.local.yml` to run every minute during testing, then revert:

```yaml
services:
  apiserver:
    environment:
      DT_TASK_DEFECT_DOJO_UPLOAD_CRON: "* * * * *"
```

The task scheduler uses `db-scheduler` with a distributed lock — only one of the three replicas runs each task at a time. Check `scheduled_tasks` in Postgres to see `last_success` and confirm a run completed.

`dev/docker-compose.local.yml` overrides the apiserver image to `dependencytrack/apiserver:local` and sets `ALPINE_BCRYPT_ROUNDS=4` for fast logins in dev. Build the image first (see Build environment above).

## DefectDojo integration

- `apiserver/src/main/java/org/dependencytrack/integrations/defectdojo/`
- `DefectDojoUploader` reads per-project `ProjectProperty` values and forwards them to `DefectDojoClient`
- `DefectDojoClient` uses `MultipartBodyPublisher` (not Apache HttpClient's `MultipartEntityBuilder` — that was v4)
- Pattern for new per-project properties: follow `defectdojo.testTitle` as the reference implementation
- Per-project properties use `groupName = DEFECTDOJO_ENABLED.getGroupName()` (value: `"integrations"`)

## Open PRs

Two PRs open against `DependencyTrack/dependency-track:main`, companion to already-submitted `4.14.x` PRs:

| PR | Title | 4.14.x companion |
|----|-------|-----------------|
| #6415 | Forward analysis.detail to DefectDojo | #6181 |
| #6416 | Forward group_by parameter to DefectDojo | #6130 |

Maintainer `nscuro` asked contributors to split v4/v5 changes by branch (`4.14.x` vs `main`) per `V5_MIGRATION.md`. The v4 PRs were retargeted to `4.14.x` and the v5 ports submitted separately to `main`.

## Branch conventions

- Branch names use hyphens, not slashes
- PRs against `main` for v5 features, against `4.14.x` for v4 backports
- Push to the fork remote, not `origin` (which points to the upstream DependencyTrack org)
