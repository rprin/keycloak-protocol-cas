# AGENTS.md

Keycloak plugin (Quarkus distribution, Keycloak 17+) adding the CAS 3.0 SSO protocol. Single-module Maven project, Java 17. Modeled on Keycloak's own OIDC implementation.

## Commands

- Build + unit tests: `mvn test package` (what CI runs; produces `target/keycloak-protocol-cas-<version>.jar`)
- Single unit test: `mvn test -Dtest=RedirectUtilsTest` (plain JUnit Jupiter + Mockito; no server needed, KeycloakSession etc. are mocked)
- No lint/format/CI typecheck steps exist.
- Unit tests validate XML responses against XSDs in `src/test/resources/org/keycloak/protocol/cas/`.

## Integration tests (separate from `mvn test`)

`integrationTest/suite.sh` is a bash+curl suite (CAS 1.0/2.0/3.0, SAML 1.1, gateway mode) that needs a live Keycloak on `localhost:8080`. The exact procedure is in `.github/workflows/test.yml`:

1. `docker run ... quay.io/keycloak/keycloak:<keycloak.version> start-dev` with `KEYCLOAK_ADMIN=admin`, `KEYCLOAK_ADMIN_PASSWORD=admin`, port 8080
2. Copy the built jar into the container at `/opt/keycloak/providers/`, restart the container
3. Via `kcadm.sh`: create client `test` with `protocol=cas`, `publicClient=true`, `redirectUris=["http://localhost/*"]`, `baseUrl`/`adminUrl=http://localhost`
4. Run `integrationTest/suite.sh` (uses cookie jar at `/tmp/cookies`)

## Versioning and releases (important)

- Plugin version **must equal** the Keycloak version: `keycloak.version` is set to `${project.version}` in `pom.xml`. When bumping for a new Keycloak release, update the project version and all dependency versions together (see the automated `update-deps.yml` workflow: `mvn versions:set -DnewVersion=<keycloak-version>` plus comparison against `org.keycloak:keycloak-parent`). Also bump `project.build.outputTimestamp` (reproducible builds).
- Releases are tag-driven: pushing a tag matching `[0-9]+.*` runs the test workflow and auto-creates a GitHub release with the jar (`release.yml`). Do not upload artifacts manually.
- Daily bot opens `feature/keycloak-update-X.Y.Z` branches against `master`; merge those after CI passes.

## Architecture gotchas

- SPI wiring is file-based in `src/main/resources/META-INF/services/`. New mappers must be added to `org.keycloak.protocol.ProtocolMapper`, and the factory/installation classes are registered there too. Forgetting the service file means Keycloak silently never sees the provider.
- Entry point: `CASLoginProtocolFactory` (protocol id `cas`). Endpoints live in `endpoints/` (login = `AuthorizationEndpoint`, plus validate/proxy/logout/SAML endpoints); attribute and role-to-CAS-assertion mappers live in `mappers/`.
- All `org.keycloak` dependencies are `provided` scope (not bundled into the jar). If you add a new non-bundled runtime dependency, also add it to the `Dependencies` manifest entry in `maven-jar-plugin` config, or Keycloak won't resolve it at load time.
- CAS clients have **no client ID in the protocol** — the client is identified by its configured redirect URIs, and the `service` parameter is validated against them.
