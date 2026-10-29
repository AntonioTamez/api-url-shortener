# URL Shortener API

A URL shortener built with .NET and deployed to Azure. It starts with a delivery platform (pipeline, scans, Azure hosting and observability) and then grows phase by phase as a controller-based API: each phase adds a business capability, a cloud/DevOps practice, or both.

> Status: planning. No code yet. See [docs/roadmap.md](docs/roadmap.md).

## Why this project

| Goal                        | How the project covers it                                                     |
| --------------------------- | ----------------------------------------------------------------------------- |
| Learn Azure end to end      | Each Azure service is added in the phase that gives it a real reason to exist |
| Practice CI/CD and security | Pipeline, tests and vulnerability scanning exist from phase 0, not at the end |
| Practice clean design       | Clean/Hexagonal Architecture keeps the domain independent of Azure            |

## Core use case

1. A user sends a long URL: `POST /links` → returns a short code.
2. Anyone opens the short URL: `GET /{code}` → `302` redirect to the long URL.

Everything else (users, stats, cache, rate limiting) builds on these two endpoints.

## Tech stack

| Area                   | Choice                                                                                                |
| ---------------------- | ----------------------------------------------------------------------------------------------------- |
| Runtime                | .NET 10 (LTS), ASP.NET Core Web API (controllers)                                                     |
| Tests                  | xUnit, `WebApplicationFactory`, Testcontainers                                                        |
| CI/CD                  | GitHub Actions                                                                                        |
| Hosting                | Azure App Service (Linux), later Azure Container Apps                                                 |
| Infrastructure as Code | Bicep                                                                                                 |
| Security scanning      | SonarQube Cloud, CodeQL, Dependabot, NuGetAudit, `dotnet list package --vulnerable`, OWASP ZAP, Trivy |

## Getting started

Follow the [Phase 0 guide](docs/guides/phase-0.md): it starts with creating the solution locally and ends with the API running in Azure.

## Documents

- [Roadmap](docs/roadmap.md): phases, features and done criteria.
- [Architecture](docs/architecture.md): structure and key decisions.
- [Phase 0 guide](docs/guides/phase-0.md): step-by-step instructions from local to Azure..
