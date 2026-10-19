# TheSwitch — Architecture Document

Oct 19, 2026

## Context and scope

TheSwitch is a mobile app whose AI agent makes the first step ridiculously easy: it decides, checks and adjusts in place of willpower.

**Essence.** An external structure that takes over when willpower isn't enough. The agent combines four modes (strategist, mirror, provocateur, judge) and picks the right one based on the user's state.

**First domain: getting started on dev work.** It tackles the inertia of getting down to work and the difficulty of starting a project that feels too big. It is measured automatically through GitHub.

**Out of scope for v1:** sport (second module), screen-time tracking (third), interchangeable narrative themes (once the engine is stable), iOS version (after the Android v1).

**Quality goal.** A production-grade app, not a prototype: layered architecture, CI/CD, agent observability, automated evals.

## Overview

The API responds fast and hands any long-running work to the worker through Redis. The worker runs the agent, calls the Anthropic API and sends notifications through FCM.

&#91;embedded content: architecture · 4 Railway services, 3 external services\]

The API and the worker share the same PostgreSQL database. Every agent run is traced in Langfuse.

## Tech stack

Everything is hosted on Railway, except push notification delivery, which must go through Firebase Cloud Messaging.

| Layer | Choice | Role |
| --- | --- | --- |
| Mobile | Flutter, Riverpod, go\_router, dio, freezed | Polished UI, state, navigation, immutable models |
| API client | Generated from the FastAPI OpenAPI schema | Front end and back end always in sync |
| API | FastAPI, Pydantic, SQLAlchemy 2, Alembic | Endpoints, validation, data access, migrations |
| Worker | arq + Redis | Daily loop, nudges, webhook processing |
| Database | PostgreSQL (Railway) | Business data and agent history |
| Agent | Anthropic SDK behind an `LLMClient` interface | Controlled tool-use loop, swappable provider |
| Push | Firebase Cloud Messaging via `firebase-admin` | Delivery to Android (iOS later) |
| Observability | Langfuse (self-hosted), Sentry, JSON logs | Agent traces, errors, costs |
| Tooling | uv, ruff, mypy strict, pytest, Docker | Quality and reproducibility |

## Data model

Nine tables are enough for v1. The user's state and every agent decision are recorded over time, which makes evals possible.

| Table | Main content | Relations |
| --- | --- | --- |
| `users` | email, password hash, time zone, schedule preferences | — |
| `oauth_accounts` | provider (GitHub), external id, encrypted token, scopes | → users |
| `device_tokens` | FCM token, platform, last used | → users |
| `goals` | long-term goal, domain, status | → users |
| `projects` | tracked project, linked GitHub repository | → users, goals |
| `quests` | title, type (start, exploration, build, recovery), difficulty, deadline, status, validation rule | → users, projects |
| `quest_events` | validation event (commit, self-report, expiry), timestamp, proof | → quests |
| `user_state` | computed state (progressing, stagnating, dropping off), streak, stats | → users |
| `agent_runs` | trigger, chosen mode, model, tools called, tokens, cost, Langfuse trace id, result | → users |

**Quest validation rule.** Stored as typed JSON, for example "at least one commit on this repository before a given time". The worker evaluates it on every webhook, without calling the LLM.

## Agent

The mode is chosen by deterministic code from `user_state`, not by the LLM: it is cheaper, testable and predictable. The LLM then generates the content within that mode.

### Execution loop

1. A trigger arrives: scheduled morning task, expired quest, weekly review, project-breakdown request.
2. The worker recomputes `user_state` and derives the mode from it.
3. The agent receives the mode's prompt, a compact user context and the list of tools allowed for that mode.
4. Tool-use loop, capped in number of turns and tokens.
5. Actions are applied to the database, the notification is sent, and the run is recorded in `agent_runs` and Langfuse.

### Modes

| Mode | Triggered when | What it produces | Model |
| --- | --- | --- | --- |
| Strategist | Every morning, project breakdown | Daily quests, exploration and build tickets | Haiku day to day, Sonnet for breakdowns |
| Mirror | Weekly review, large gap between goal and actual | Fact-based report, no penalty | Sonnet |
| Provocateur | Stagnation: quests passed but too easy several days in a row | A challenge that breaks the routine | Haiku |
| Judge | Dropping off: several days without activity | A single short, non-negotiable quest | Haiku |

### Tools

- `get_user_state`: state, streak, stats.
- `get_github_activity`: recent commits on linked repositories.
- `list_quests`: ongoing quests and recent history.
- `create_quest`: creates a quest with its validation rule.
- `update_difficulty`: adjusts the difficulty level.
- `split_project`: proposes a ticket breakdown for a project.
- `send_notification`: schedules a nudge.

Each tool is a Pydantic model. The same schema serves as the definition sent to the LLM and as the validation of the call.

### Guardrails

- No more than one active recovery quest at a time: the judge restarts, it doesn't pile on.
- Token budget per run and per user per day.
- No destructive action available to the agent.

## Main flows

### Morning start quest

1. At the time chosen by the user, arq runs the scheduled task.
2. The agent in strategist mode creates a tiny quest: "sneakers on, editor open, one commit within 30 minutes".
3. The worker sends the notification through FCM.
4. If there is no commit by the deadline, the quest expires and an expiry `quest_event` is recorded. The state is recomputed on the next trigger.

### GitHub webhook validation

1. GitHub sends a `push` event to the API.
2. The API checks the HMAC signature, stores the event and responds immediately.
3. The worker evaluates the validation rules of open quests, without the LLM.
4. A validated quest updates the stats and triggers a confirmation notification.

### Device registration

1. The Flutter app fetches an FCM token at launch.
2. It sends it to `POST /devices`, which stores it in `device_tokens`.
3. A token rejected by FCM is deleted automatically.

## Auth, security and observability

### Authentication

- GitHub OAuth sign-in, which serves as both login and integration. Optional email and password, hashed with argon2.
- Short-lived JWT access token and rotating refresh token, stored on the device in the system's secure storage.

### Security

- GitHub tokens encrypted in the database.
- HMAC signature checked on every webhook.
- Secrets only in Railway environment variables, never in the repo.
- Rate limiting on auth endpoints.

### Observability

- **Langfuse**: one trace per agent run, with prompts, tool calls, tokens and cost.
- **Sentry**: errors from the API, the worker and Flutter.
- **JSON logs** with a request id propagated from the API to the worker.
- **Evals**: a set of scenarios (user dropping off for three days, stagnation, first week) that checks the chosen mode and the shape of generated quests.

## Repo, CI/CD and environments

A single monorepo, with a CI that only runs the jobs for changed folders.

```
theswitch/
├── apps/
│   ├── api/          # FastAPI + arq worker (two entry points, one package)
│   └── mobile/       # Flutter
├── packages/
│   └── api_client/   # Dart client generated from the OpenAPI schema
├── evals/            # agent evaluation scenarios
├── infra/            # local docker-compose, Railway config
├── docs/             # ADRs and documentation
└── .github/workflows/
```

### Pipeline

1. On every pull request: ruff, mypy and pytest for the API; `flutter analyze` and `flutter test` for mobile; a check that the generated client is up to date.
2. On merge to `main`: automatic deployment of the API and worker to staging.
3. Tagged release: deployment to production, once the evals pass.

### Environments

| Environment | API and worker | Data | Use |
| --- | --- | --- | --- |
| Local | docker-compose | Postgres and Redis in containers | Development |
| Staging | Railway | Dedicated Postgres and Redis | Testing on a real device |
| Production | Railway | Dedicated Postgres and Redis | Daily use |

Structural decisions are recorded as ADRs in `docs/`, as a team would do.

## Open questions and next steps

### Open questions

- Final name of the project and repository: TheSwitch, to evoke change (tentative, repository `theswitch`).
- 3 days before considering the user in "dropdown"
- accepts automatic proof only in order to keep the work steady and concrete. Ideally making a small review of what was done today to motivate the user should be considered in the next iterations.
- v1 target: Android only (decided on 2026-10-19).

### Next steps

- [x] Create the GitHub repository and the monorepo structure
- [ ] Write the first ADR (stack choice)
- [ ] Set up local docker-compose and a minimal CI
- [ ] Build GitHub OAuth authentication
- [ ] Build the agent loop in strategist mode only, with one start quest
- [ ] Wire up the GitHub webhook and automatic validation
