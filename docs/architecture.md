# Architecture

The domain (links, codes, rules) must not depend on ASP.NET, EF Core or Azure. Azure services are adapters that can be replaced without touching business rules.

## Solution structure

```text
src/
  UrlShortener.Domain/          # Entities, value objects, domain rules. No dependencies.
  UrlShortener.Application/     # Use cases and ports (interfaces). Depends on Domain.
  UrlShortener.Infrastructure/  # Adapters: SQL, Redis, Service Bus. Implements ports.
  UrlShortener.Api/             # Controllers, auth, DI wiring. Composition root.
  UrlShortener.StatsFunction/   # Azure Function for click events (phase 4).
tests/
  UrlShortener.Domain.Tests/
  UrlShortener.Application.Tests/
  UrlShortener.Api.IntegrationTests/
infra/                          # Bicep templates (phase 0).
.github/workflows/              # CI/CD pipelines.
.github/actions/deploy/         # Composite action shared by the dev and prod deploy jobs.
.github/dependabot.yml          # Dependency update configuration.
.zap/rules.tsv                  # OWASP ZAP alerts accepted on purpose, each with its reason.
```

Folders inside `Application` are organized by feature (`Links/CreateLink`, `Links/ResolveLink`, `Stats/...`), so the structure says what the system does, not which framework it uses.

## Dependency rule

```text
Api ──► Application ──► Domain
 │            ▲
 └──► Infrastructure
```

`Infrastructure` implements interfaces defined in `Application`. `Domain` references nothing.

## Key decisions

| Topic | Decision | Reason |
|-------|----------|--------|
| API style | Controllers | Familiar MVC structure; actions grouped per feature controller |
| Short code | 7 random Base62 characters | ~3.5 trillion combinations; not guessable like sequential IDs |
| Redirect status | `302` | `301` is cached by browsers, which would hide clicks from stats |
| Errors | RFC 9457 Problem Details | Standard, consistent error bodies |
| Secrets | Managed Identity + Key Vault | No application passwords or connection-string secrets in config or pipelines |
| Pipeline auth to Azure | OIDC federated credentials | No long-lived Azure credentials in GitHub |
| Stats write path | Async through a queue | Redirect latency must not depend on stats |
| Versioning | URL prefix `/v1` from phase 6 | Simple and visible for clients |

## Open questions

- Azure SQL vs Cosmos DB: SQL is the default; reconsider if read volume makes key-value access dominant.
- Entra ID vs Entra External ID: depends on whether public users can sign up.
