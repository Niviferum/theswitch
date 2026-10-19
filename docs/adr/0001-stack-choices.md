# ADR 0001 — Stack choices

- **Status:** Accepted
- **Date:** 2026-10-19
- **See also:** [Architecture document](../architecture.md)

## Context

TheSwitch is a mobile app whose AI agent helps its user get started on dev work by deciding, checking and adjusting in place of willpower. The project has two goals of equal weight:

1. **Build a skill.** Learn agentic development in depth, with solid foundations rather than abstractions I don't understand.
2. **Build an app I will actually use.** Production-grade, not a prototype, with a front end I am proud of.

Constraints: solo developer, Android only for v1, a small infrastructure budget, and as little operational sprawl as possible.

## Decision

| Area | Choice |
| --- | --- |
| Mobile | Flutter (Riverpod, go_router, dio, freezed), Dart client generated from the OpenAPI schema |
| Backend | Python: FastAPI, Pydantic, SQLAlchemy 2, Alembic |
| Background jobs | arq + Redis |
| Database | PostgreSQL |
| Agent | No agent framework: a hand-written tool-use loop on the Anthropic SDK, behind an `LLMClient` interface |
| Hosting | Railway for everything (API, worker, PostgreSQL, Redis, Langfuse) |
| Push | Firebase Cloud Messaging, the only service outside Railway |

### Rationale

**No agent framework.** The point of the project is to learn agentic development, so the core mechanics are written by hand: the tool-use loop, tool schemas, turn and token limits, mode selection, tracing and evals. A framework would hide exactly what I want to master. The `LLMClient` interface keeps the provider swappable without depending on a framework.

**Flutter.** It gives full control over rendering and animation, which lets me push the UI much further than standard native components. The game-like "System" interface is a core part of the experience, and I want a front end I'm proud of. It also keeps the door open for iOS later from the same codebase.

**Python backend.** Python is one of the most widely used languages in the agentic ecosystem today: official SDKs, tooling, observability and evaluation libraries all target it first. Working in it builds directly on the skill this project is meant to develop. FastAPI and Pydantic provide typed schemas that serve as API contracts, tool definitions sent to the LLM, and validation of tool calls.

**Everything on Railway.** Hosting the API, worker, database, Redis and Langfuse on a single platform keeps the app centralised: one dashboard, one place for secrets and environment variables, the same setup for staging and production. FCM is the one exception, because Android push delivery must go through Google.

**arq + Redis.** It is async-native like FastAPI, lightweight, and covers what the worker needs: scheduled tasks, deferred jobs and webhook processing.

## Alternatives considered

| Alternative | Why it was rejected |
| --- | --- |
| LangChain, LangGraph, CrewAI or another agent framework | Hides the mechanics I want to learn, and adds abstraction and dependency churn for a loop that stays small |
| Supabase or Firebase as the backend | Splits the infrastructure across providers and hands the core backend to a BaaS; I want to own and understand the backend |
| React Native or Kotlin native | React Native: less control over custom rendering than Flutter. Kotlin: Android only, which rules out a cheap iOS port |
| Node.js / TypeScript backend | A valid choice, but Python is more central to the agentic ecosystem I am training in |
| Celery | Heavier and sync-oriented; arq fits an async FastAPI codebase better |

## Consequences

**Positive**
- Deep understanding of how agents work, which can be shown in interviews through the code, traces and evals.
- Full control over agent behaviour, cost and guardrails.
- A single hosting platform and a single backend language keep operations simple for a solo developer.
- Typed contracts end to end: Pydantic → OpenAPI → generated Dart client.

**Negative / risks**
- More code to write and maintain than with a framework (retries, context handling, tracing). This is mitigated by keeping the loop small and covering it with tests and evals.
- Two languages (Dart and Python) to master in parallel.
- Dependence on Railway: pricing changes or outages affect every service at once. Docker images and a local docker-compose keep a migration feasible.
- FCM adds a Google dependency and its own credentials to manage.
