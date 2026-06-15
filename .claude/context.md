# Dependency-Track v5 — Agent Context

## Build environment

The project targets Java 25 (`<release>25</release>` in root `pom.xml`).
On this machine, Java 25 is installed as a **JRE only** — `javac` is missing.
Maven falls back to the Java 21 JDK, which fails to compile the project.

**Fix:** install the Java 25 JDK and set JAVA_HOME before running any make/mvn commands.

```bash
sudo apt install openjdk-25-jdk
export JAVA_HOME=/usr/lib/jvm/java-25-openjdk-amd64
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

## DefectDojo integration

- `apiserver/src/main/java/org/dependencytrack/integrations/defectdojo/`
- `DefectDojoUploader` reads per-project `ProjectProperty` values and forwards them to `DefectDojoClient`
- `DefectDojoClient` uses `MultipartBodyPublisher` (not Apache HttpClient's `MultipartEntityBuilder` — that was v4)
- Pattern for new per-project properties: follow `defectdojo.testTitle` as the reference implementation
