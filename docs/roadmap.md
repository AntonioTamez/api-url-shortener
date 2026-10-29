# Roadmap

Each phase is a deployable increment: it ends with the API running in Azure, the pipeline green, and the scans clean. Do not start a phase until the previous one is done.

## Overview

| Phase | Feature | Azure / DevOps added |
|-------|---------|----------------------|
| 0 | Delivery platform (`/health` only) | GitHub Actions, SonarQube, CodeQL, Dependabot, Bicep, App Service, OIDC, Application Insights, OWASP ZAP, dev/prod with approvals |
| 1 | Create and resolve links (in memory) | — (uses the Phase 0 pipeline) |
| 2 | Persistence | Azure SQL, Key Vault, Managed Identity |
| 3 | Users own their links | Microsoft Entra ID, JWT auth |
| 4 | Click statistics | Service Bus, Azure Functions |
| 5 | Fast redirects | Azure Cache for Redis |
| 6 | Expiration, custom aliases, rate limiting | API Management |
| 7 | Business observability | Custom metrics, dashboards |
| 8 | Containers | Container Apps, ACR, Trivy |

---

## Phase 0: Delivery platform

**Goal:** the full path from local to Azure works before any business feature exists. Every later phase builds on a pipeline that is already green, scanned and observable.

**Steps**

| # | Step | Includes |
|---|------|----------|
| 1 | Local solution | Solution scaffold (see the [Phase 0 guide](guides/phase-0.md)), `global.json`, analyzers, GitHub repo, branch ruleset on `main` (PR required, checks must pass) |
| 2 | API skeleton | `GET /health` for liveness, OpenAPI document, RFC 9457 Problem Details, unit test project in the pipeline (real domain tests start in Phase 1), integration tests for `/health` (`200`) and an unknown route (`404` as Problem Details) |
| 3 | CI | GitHub Actions: restore → `dotnet format --verify-no-changes` → build → test with coverage (coverlet) |
| 4 | Code quality | SonarQube Cloud with coverage import; the quality gate is a required PR check |
| 5 | SAST / SCA | CodeQL on every PR, Dependabot for NuGet and GitHub Actions, NuGetAudit, secret scanning with push protection |
| 6 | Infrastructure as code | Bicep for App Service plan, App Service (Linux), Log Analytics, Application Insights; the resource group is created by hand and is the deployment target |
| 7 | Pipeline access to Azure | OIDC federated credentials; no Azure credentials stored in GitHub |
| 8 | CD to `dev` | Pipeline deploys Bicep, then the app; smoke test against `/health` |
| 9 | Baseline observability | OpenTelemetry exporting to Application Insights, structured logging, correlation IDs |
| 10 | DAST | OWASP ZAP baseline scan against `dev`; fail on any alert not explicitly accepted in `.zap/rules.tsv` |
| 11 | `prod` environment | GitHub Environments with manual approval; optional deployment slot with swap |
| 12 | Alerts | Alerts on failed requests and latency |

**Done when**
- [ ] A merge to `main` deploys to `dev` without manual steps, and to `prod` after approval
- [ ] Infrastructure can be recreated with Bicep after recreating the resource group and the pipeline's role assignment
- [ ] Unit and integration tests run in the pipeline, with coverage visible in SonarQube
- [ ] SonarQube quality gate passes; CodeQL and Dependabot report no open high/critical findings
- [ ] ZAP scan passes: every alert is fixed or accepted with a written reason
- [ ] Requests to `dev` appear in Application Insights with a correlation ID

## Phase 1: Walking skeleton

**Goal:** the smallest useful API, shipped through the Phase 0 pipeline.

**Features**
- `POST /links` with `{ "url": "https://..." }` → `201` with `{ "code": "aB3x9Zq", "shortUrl": "..." }`
- `GET /{code}` → `302` redirect, or `404` when the code does not exist
- Validate input: absolute `http`/`https` URL only → `400` otherwise

**Azure / DevOps**
- Nothing new: the Phase 0 pipeline builds, scans and deploys these endpoints

**Done when**
- [ ] Unit and integration tests cover `201`, `302`, `404` and `400`
- [ ] The endpoints run in `prod` with the quality gate and scans clean

## Phase 2: Persistence

**Goal:** links survive restarts.

**Features**
- Store links in a database through a repository port (the domain does not know SQL)
- Unique index on `code`; retry on collision

**Azure / DevOps**
- Azure SQL Database (serverless tier to save cost)
- Managed Identity from App Service to SQL: no connection-string passwords
- Key Vault for any remaining secrets
- Add the new resources to the Phase 0 Bicep templates
- EF Core migrations applied as a pipeline step
- Integration tests against a real SQL Server with Testcontainers

**Done when**
- [ ] Infrastructure can be recreated with Bicep after recreating the resource group and the pipeline's role assignment
- [ ] No secret exists in code or config files, and no Azure credential exists in GitHub

## Phase 3: Users and ownership

**Goal:** authenticated users manage their own links.

**Features**
- Creating links requires authentication; redirects stay public
- `GET /me/links`, `DELETE /links/{code}` (owner only → `403` otherwise)

**Azure / DevOps**
- Microsoft Entra ID (or Entra External ID for public sign-up)
- JWT bearer validation in the API

**Done when**
- [ ] Tests cover `401` (no token) and `403` (not owner)

## Phase 4: Click statistics

**Goal:** owners see how their links perform, without slowing down redirects.

**Features**
- Every redirect publishes a `LinkClicked` event (time, referrer, country, user agent)
- `GET /links/{code}/stats` → totals per day, country and referrer

**Azure / DevOps**
- Azure Service Bus queue for click events
- Azure Function consumes events and writes aggregates
- Separate pipeline job for the Function

**Project setup**

```bash
dotnet new install Microsoft.Azure.Functions.Worker.ProjectTemplates
dotnet new func --name UrlShortener.StatsFunction --output src/UrlShortener.StatsFunction
dotnet sln add src/UrlShortener.StatsFunction --solution-folder src
dotnet add src/UrlShortener.StatsFunction reference src/UrlShortener.Application src/UrlShortener.Infrastructure
```

**Done when**
- [ ] A redirect does not wait for the stats write
- [ ] A failed event ends in the dead-letter queue, not lost

## Phase 5: Fast redirects

**Goal:** redirects served from cache.

**Features**
- Cache-aside on `GET /{code}`; invalidate on delete

**Azure / DevOps**
- Azure Cache for Redis
- Load test (Azure Load Testing or k6) comparing before and after

**Done when**
- [ ] Load test shows the improvement with numbers

## Phase 6: Business rules and protection

**Features**
- Optional expiration date → `410 Gone` after it
- Custom aliases (`/my-event`) with reserved-word list
- Rate limiting per user and per IP on `POST /links`
- Malicious-URL check before accepting a link (e.g. Google Safe Browsing)

**Azure / DevOps**
- Azure API Management in front of the API: rate-limit policies, subscription keys, versioning (`/v1`)

## Phase 7: Business observability

Baseline telemetry, structured logging and alerts already exist from Phase 0.

**Features**
- Custom metrics: links created, redirects, cache hit ratio

**Azure / DevOps**
- A dashboard in Azure Monitor with business and technical metrics
- Alerts on business signals (e.g. redirect failures, cache hit ratio drop)

## Phase 8: Containers

**Features**
- No new business features: this phase is about delivery

**Azure / DevOps**
- Dockerfile; image pushed to Azure Container Registry
- Trivy image scan in the pipeline; fail on high/critical
- Move to Azure Container Apps, keeping the Phase 0 `dev`/`prod` environments and approvals

## Future ideas

- QR code per link
- Bulk link creation
- Webhooks when a link reaches N clicks
- Admin endpoints and abuse reporting
