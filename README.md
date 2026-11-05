# Case studies

Decision records from systems I've designed. Architecture and trade-offs only — names,
schemas and figures are deliberately omitted; the reasoning is mine to discuss, the data is not.

| Case study | In one line |
|---|---|
| [Moving a live commerce platform between two databases without a maintenance window](migration-live-commerce-platform.md) | Six product lines, two databases both serving throughout, two flows kept rigorously separate, and the reconciliation lesson I'd carry into any dual-write system |

## Reference code

Small, tested sample projects for the patterns these records rely on. They are written for this
portfolio and contain no employer code.

| Repository | Pattern |
|---|---|
| [db-migration-reconciler](https://github.com/hardyalsinghdeora/db-migration-reconciler) | The reconciliation step of a migration with no maintenance window: keyset-chunked checksums, drill-down diffs and a dry-run-first, resumable repair |
| [event-outbox-worker](https://github.com/hardyalsinghdeora/event-outbox-worker) | Transactional outbox, at-least-once relay and idempotent consumer |
| [spring-boot-order-service](https://github.com/hardyalsinghdeora/spring-boot-order-service) | Order-to-cash API in Java / Spring Boot: idempotency keys, `If-Match` and an explicit state machine |
| [api-gateway-rate-limiter](https://github.com/hardyalsinghdeora/api-gateway-rate-limiter) | Rate limiting, circuit breaking and safe retries at the edge |

## Profile

| File | What it is |
|---|---|
| [Portfolio](https://hardyalsinghdeora.pages.dev/) | Interactive portfolio page — ask it questions about my work ([source](https://github.com/hardyalsinghdeora/portfolio)). `portfolio.html` here only redirects there. |
| [resume.html](https://hardyalsinghdeora.github.io/case-studies/resume.html) | One-page résumé (print to PDF from the browser) ([source](resume.html)) |
| [profile.json](profile.json) | Machine-readable career profile — experience, systems owned, skills, current focus, recruiter quick answers. A copy of the portfolio's [`knowledge/profile.json`](https://github.com/hardyalsinghdeora/portfolio/blob/main/knowledge/profile.json); CI fails if the two differ |

Contact: hardyalsinghdeora@gmail.com · [LinkedIn](https://www.linkedin.com/in/hardyalsinghdeora)
