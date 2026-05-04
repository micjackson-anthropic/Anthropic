# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Workspace Overview

This is a multi-project consultant workspace, not a single repository. Projects live primarily under `Documents\Projects\` (40+ client engagements) and `Downloads\`. The primary tech stacks are:

- **SAP Hybris DataHub** (Java 1.8 / Maven / Spring 4.2) — master data management pipelines
- **TIBCO BusinessWorks 5.15** — event-driven middleware/integration (`.process` files, XML config)
- **MuleSoft Anypoint** (Mule runtime 4.11.2, Java 17) — integration platform

## Key Project Locations

| Path | Description |
|------|-------------|
| `Documents\Projects\Alcon\ALCON - DESKTOP - CONTENT\code\AlconInvoiceHistory\` | SAP Hybris DataHub invoice history pipeline (3-module Maven) |
| `Documents\Projects\Alcon\ALCON - DESKTOP - CONTENT\code\csv-file-feed-service\` | Hybris DataHub CSV import REST service |
| `Downloads\TMS_Code\TMS_RT_LoadDetails-bw515\` | TIBCO BW 5.15 — TMS load tender event processing |
| `AnypointCodeBuilder\` | MuleSoft Anypoint Code Builder workspace (VS Code-based) |
| `AnypointStudio\` | MuleSoft Anypoint Studio workspace |

## Build & Test Commands

### Java/Maven (Hybris DataHub projects)
```powershell
# Build from project root (where pom.xml lives)
mvn clean install

# Skip tests (tests are skipped by default in these projects via pom.xml)
mvn clean install -DskipTests

# Run a specific test class
mvn test -Dtest=ClassName

# Build a single module in a multi-module project
mvn clean install -pl AlconInvoiceHistory-canonical
```

Maven local repository is at `~\.m2\repository`. Java 1.8 is required for DataHub projects (`maven.compiler.source/target=1.8`).

### TIBCO BusinessWorks
TIBCO BW projects (`.process` files) are deployed via TIBCO Administrator or BW Designer IDE — there is no CLI build step. Deployment configs are environment-specific `.properties` files under `app.bw\TMS_RT_LoadDetails\Deployments\`. Passwords are TIBCO-encrypted (prefixed with `#!`).

### MuleSoft Anypoint
Projects are opened and run from AnypointCodeBuilder or AnypointStudio. The Mule 4.11.2 runtime requires **Java 17** (bundled at `AnypointCodeBuilder\java\`). MCP auth token is at `~\.mulesoft-mcp\mcp_auth_token`.

## Architecture Patterns

### Hybris DataHub 3-Tier Pipeline
All DataHub extensions follow a raw → canonical → target pattern, each as a separate Maven module:
- **Raw module**: Plain Java domain class matching source data exactly — no transformation
- **Canonical module**: Business rule transformations via `CompositionHandler` implementations (e.g., `JulianDateCompositionHandler` converts `Myydd` → ISO 8601)
- **Target module**: Final domain class mapped to destination system format

The parent POM aggregator (`pom.xml` with `packaging=pom`) declares all three as modules. Key dependency: `datahub-extension-sdk` v5.6.0.0 (provided scope).

### CSV Import Service
`csv-file-feed-service` exposes `POST /file-data-feeds/{feedName}/items/{type}` backed by:
- Spring XML beans in `META-INF/csv-file-feed-service-datahub-extension-spring.xml` — file-type-to-header mappings keyed by bean name (e.g., `BRFAAEC1U`)
- Background `DirectoryListenerThread` polling for new files; atomic moves to archive/error dirs
- Batch size: 10,000 rows; empty CSV fields represented as `<empty>` string marker
- **Legacy stack**: Jersey 1.19, Jackson 1.9.2, Spring 4.2.1 — do not attempt to upgrade without full regression testing

### TIBCO BW Event Processing
Timer-triggered starter processes (`TenderAccepted.process`, `TenderUpdated.process`) run on 60-second intervals using global variable substitution (`%%namespace/varName%%`). All config (file paths, EMS URLs, Oracle JDBC strings, iAPI identifiers) is injected at deploy time via per-environment `.prop` files — there is no runtime override mechanism. Shared logic lives in `TMSCommonLib_v2.0.0.projlib` and `main-iapi-bw-0.3.6.projlib` (Macy's internal iAPI framework).

## Key Gotchas

- **Java version split**: DataHub projects require Java 1.8; Mule 4.11.2 requires Java 17. Confirm `JAVA_HOME` before switching between them.
- **No unit tests**: Both Alcon DataHub projects have `skipTests=true` hardcoded in their POMs. Manual/integration testing is the norm.
- **local.properties has hardcoded paths**: `C:\datahub56\testFiles_*` — override per environment, never commit dev-local paths.
- **csv-file-feed-service references `../parent/pom.xml`**: Verify the parent POM exists relative to the project before building.
- **TIBCO library portability**: `.projlib` file aliases may not resolve on machines where TIBCO Designer paths differ.
