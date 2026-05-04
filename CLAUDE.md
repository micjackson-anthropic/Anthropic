# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Workspace Overview

This is a multi-project consultant workspace, not a single repository. Projects live primarily under `Documents\Projects\` (40+ client engagements) and `Downloads\`. The primary tech stacks are:

- **SAP Hybris DataHub** (Java 1.8 / Maven / Spring) — master data management pipelines
- **TIBCO BusinessWorks 5.15** — event-driven middleware/integration (`.process` files, XML config)
- **MuleSoft Anypoint** — integration platform (AnypointCodeBuilder, AnypointStudio)

## Key Project Locations

| Path | Description |
|------|-------------|
| `Documents\Projects\Alcon\` | SAP Hybris DataHub extensions (CSV import, invoice history) |
| `Downloads\TMS_Code\` | TIBCO BW 5.15 — TMS load tender event processing |
| `AnypointCodeBuilder\` | MuleSoft Anypoint Code Builder workspace |
| `AnypointStudio\` | MuleSoft Anypoint Studio workspace |

## Build & Test Commands

### Java/Maven (Hybris DataHub projects)
```powershell
# Build from project root (where pom.xml lives)
mvn clean install

# Skip tests
mvn clean install -DskipTests

# Run a specific test class
mvn test -Dtest=ClassName

# Build a single module in a multi-module project
mvn clean install -pl <module-name>
```

Maven local repository is at `~\.m2\repository`.

### TIBCO BusinessWorks
TIBCO BW projects (`.process`, `.bwp`) are deployed via TIBCO Administrator or the BW Designer IDE. There is no CLI build step — deployment configs live in `Deployments\` folders as environment-specific `.properties` files.

### MuleSoft Anypoint
Projects are opened and run from AnypointCodeBuilder (VS Code-based) or AnypointStudio. MCP auth token is stored at `~\.mulesoft-mcp\mcp_auth_token`.

## Architecture Patterns

### Hybris DataHub 3-Tier Pipeline
All DataHub extensions follow a raw → canonical → target pattern, each as a separate Maven module:
- **Raw module**: Plain Java domain class matching source data (no transformation)
- **Canonical module**: Business rule transformations via `CompositionHandler` implementations (e.g., date format conversion, value normalization)
- **Target module**: Final domain class mapped to the destination system format

Example: `AlconInvoiceHistory-raw` → `AlconInvoiceHistory-canonical` → `AlconInvoiceHistory-target`

Parent POM is typically `com.alcon.datahub:datahub-parent` or equivalent client artifact.

### CSV Import Service Pattern
The `csv-file-feed-service` pattern exposes a REST endpoint (`/file-data-feeds/{feedName}/items/{type}`) backed by:
- A facade layer orchestrating import flow
- Background directory-watcher threads for file-based ingestion
- Configurable CSV delimiters

### TIBCO BW Event Processing
Starter processes (`TenderAccepted.process`, `TenderUpdated.process`) are triggered by Timer or JMS sources, generate transaction IDs, invoke sub-processes for file handling, then call mapping processes. Environment configs are in `Deployments\` and shared libraries are referenced as `.projlib` files.
