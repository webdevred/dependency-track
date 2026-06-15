# Dependency-Track v5 — Agent Context

## Build environment

Requires **JDK 25** and **Maven 4**. The Temurin distribution is recommended (see `DEVELOPING.md`).

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

# Build Docker image
$MVN -B -q -Pquick package -pl apiserver -am
docker build -f apiserver/src/main/docker/Dockerfile -t dependencytrack/apiserver:local apiserver/
```

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

- **DT UI:** http://localhost:8080 — credentials `admin / admin`
- **DefectDojo UI:** http://localhost:8082 — credentials `admin / admin`
- DefectDojo takes ~2 min on first run before the UI is usable
- Get DefectDojo API token at http://localhost:8082/api/key-v2
- Configure DT→DD integration in DT admin panel:
  - URL: `http://defectdojo-nginx:8080` (Docker-internal hostname)
  - API Key: token from above

`dev/docker-compose.local.yml` overrides the apiserver image to `dependencytrack/apiserver:local` and sets `ALPINE_BCRYPT_ROUNDS=4` for fast logins in dev. Build the image first (see Build environment above).

## DefectDojo integration

- `apiserver/src/main/java/org/dependencytrack/integrations/defectdojo/`
- `DefectDojoUploader` reads per-project `ProjectProperty` values and forwards them to `DefectDojoClient`
- `DefectDojoClient` uses `MultipartBodyPublisher` (not Apache HttpClient's `MultipartEntityBuilder` — that was v4)
- Pattern for new per-project properties: follow `defectdojo.testTitle` as the reference implementation
