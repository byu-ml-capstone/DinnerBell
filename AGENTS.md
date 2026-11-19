# AGENTS.md

Dinner Bell curates recipes into a weekly meal plan fitted to a household's dietary restrictions, preferences, and budget, then turns the approved plan into a price-aware shopping list. The pilot is a Flutter web app (phone-sized layouts first) backed by Django, Django Ninja, PostgreSQL, and Celery.

## Read First

| File | When to read it |
|------|-----------------|
| `architecture.md` | **Before writing or reviewing any code.** Layers, rules, non-negotiables, Pre-Merge Checklist, Decision Log. |
| `docs/spec/dinner-bell-pilot-spec.md` | Before working on a feature: read the sections for the behavior you're touching. The spec is the source of truth for what the app does. |
| `docs/logic/<feature>.md` | Before changing business logic: the existing flowchart and logic notes for that feature. `docs/logic/README.md` has the template for new ones. |
| `audits.md` | Only when asked to review or audit code. |

**When sources disagree:** The user's task instructions take priority over this file, except for the Non-negotiable rules in `architecture.md`. If the spec and `architecture.md` conflict, stop and ask; don't pick one yourself.

## Repository Layout

| Path | Contents |
|------|----------|
| `backend/` | Django project. One app per area (`accounts`, `households`, `dietary`, `recipes`, `pricing`, `planning`, `shopping`, `feedback`, `evaluation`, `jobs`). |
| `backend/<app>/` | Any of `api.py`, `schemas.py`, `use_cases.py`, `domain.py`, `repositories.py`, `models.py`, `tasks.py`, `providers.py`, `adapters/`, `tests/` (see Stack Mapping in `architecture.md`). Create only the files an app needs; don't add empty modules or interfaces ahead of need. |
| `frontend/` | Flutter app. `lib/features/<feature>/`, `lib/core/api/`, `lib/platform/`, `test/`, `integration_test/` |
| `docs/spec/` | Product spec (the source of truth). The spec cites other team documents (*Spec Disagreements to Decide*, *Web First or Mobile First*, the spec feedback doc). They aren't in the repo, and their decisions are already folded into the spec, so you don't need them to build. |
| `docs/logic/` | Flowcharts and logic notes for business logic, one file per feature |
| `architecture.md`, `audits.md`, `AGENTS.md` | Agent and engineering guidance (repo root) |

## Setup and Commands

**TODO:** Confirm these once the repo is scaffolded. If a command is missing, marked TODO, or fails for environment reasons, tell the user instead of guessing a replacement.

**Services:** PostgreSQL 18 and Redis 8 must be running. How to start them locally: TODO (for example, `docker compose up -d`).

**Backend** (run from `backend/`):

| Task | Command |
|------|---------|
| Install dependencies | TODO |
| Apply migrations | `python manage.py migrate` |
| Run the API | `python manage.py runserver` |
| Run a Celery worker | `celery -A TODO worker -l info` |
| Run tests | `python manage.py test` (TODO: or `pytest` if the team chooses pytest-django) |
| Check for missing migrations | `python manage.py makemigrations --check --dry-run` |
| Lint and format | TODO |

**Frontend** (run from `frontend/`):

| Task | Command |
|------|---------|
| Install dependencies | `flutter pub get` |
| Run the web app | `flutter run -d chrome` |
| Analyze | `flutter analyze` |
| Format | `dart format .` |
| Run tests | `flutter test` |

## How to Work

- **Stay in scope.** Change only what the task needs. Don't refactor unrelated code; mention problems you notice instead.
- **Plan multi-file changes first.** Before editing, list which layers and files the change touches, using the layer model in `architecture.md`.
- **Don't settle open questions silently.** Spec §16 and the TODOs in `architecture.md` are undecided. Build behind an interface with a fixture or stub adapter, or stop and ask. Never hard-code a guess as if it were decided.
- **Follow Proposed items.** Items marked **Proposed** in `architecture.md` are the current default until the team confirms or replaces them. If one blocks your task or seems wrong, say so instead of working around it.
- **Record decisions in the right place.** Architecture decisions and lasting deviations go in the Decision Log in `architecture.md`. Choices and assumptions behind a feature's business logic go in `docs/logic/<feature>.md`. Don't edit the spec, including its §17 decision log, unless the task asks you to.
- **Dependencies:** Don't add, remove, or upgrade packages (pip or pub) without approval. Versions are pinned on purpose.
- **Migrations:** Generate them with `makemigrations` in the same change as the model change. Never edit or delete a migration that has been applied or merged.
- **Secrets:** Never commit credentials, API keys, or real `.env` files. Add new settings to `.env.example` with a placeholder value.
- **Generated files:** Don't hand-edit generated code (for example `*.g.dart`, `build/`). Change the source and regenerate.
- **No destructive actions** without explicit approval: dropping or resetting databases, deleting data, force-pushing, rewriting git history, deleting branches.

## Code Conventions

- **Python:** Type hints on public functions. Name use cases and state-changing functions for what they do (`approve_meal_plan`, not `update_meal_plan`).
- **Use cases and repositories:** Use cases take repositories and providers as keyword-only parameters and never import concrete ones. Repositories return data classes or plain values, never model instances. See "Repositories and use cases" in `architecture.md`.
- **Dart:** Code passes `flutter analyze` and is formatted with `dart format`.
- **API:** Routes live under `/api/v1/`. Every request and response uses a Django Ninja schema. Error response shape: TODO (define once and reuse).
- **Comments** explain non-obvious decisions, not what the code already says.

## Testing

- Every behavior change ships with tests that would fail without it. Bug fixes get a regression test.
- Unit-test domain logic without the database. Unit-test use cases with fake repositories and fake providers passed in as arguments, never patched in. Use-case tests run with database access so `transaction.atomic()` works. Test rollback behavior in integration tests with real repositories.
- Dietary logic tests include the hard cases in spec §5.4.
- Run the full test suite for each side you changed. If you change the API contract, run both backend and frontend tests.
- Never weaken, skip, or delete a test to make a change pass. If a test seems wrong, say so.

## Definition of Done

A task is done when:

1. Tests, analyzers, and the missing-migration check pass.
2. The Pre-Merge Checklist in `architecture.md` is satisfied.
3. Business logic changes include an updated flowchart and logic note in `docs/logic/`.
4. Your final summary includes:
   - what changed and why, and which files
   - which commands you ran and their results
   - any deviations from `architecture.md` and the reason for each
   - assumptions you made and open questions for the team

## Commits and Pull Requests

TODO: confirm the team's branch and review workflow. Until then:

- One logical change per commit, with a short imperative subject line ("Add restriction check to plan generation").
- Pull request descriptions use the summary format from Definition of Done.

## Stop and Ask When

- The spec is silent or ambiguous about behavior users will see.
- The spec and `architecture.md` conflict.
- A task would break a Non-negotiable rule in `architecture.md`.
- A change needs a new dependency, or a migration that changes or deletes existing data.
- A change touches authentication, authorization, or dietary restriction logic beyond what the task describes.
