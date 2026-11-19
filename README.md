# Dinner Bell

Dinner Bell curates recipes into a weekly meal plan that fits a household's dietary restrictions, preferences, and budget, then turns the approved plan into a price-aware shopping list. **Plan it · Shop it · Cook it.**

The pilot is a Flutter web app designed for phone screens, backed by Django, Django Ninja, PostgreSQL, and Celery. The project is in its design phase; the backend and frontend haven't been scaffolded yet.

## Documentation

| File | What it's for |
|------|---------------|
| [`docs/spec/dinner-bell-pilot-spec.md`](docs/spec/dinner-bell-pilot-spec.md) | What the app does: the source of truth for behavior, open questions (§16), and product decisions (§17). |
| [`architecture.md`](architecture.md) | How the code is structured: layers, non-negotiable rules, the Pre-Merge Checklist, and the architecture Decision Log. Read it before writing or reviewing code. |
| [`AGENTS.md`](AGENTS.md) | Working rules for AI coding agents. Setup commands, code conventions, and the Definition of Done live here, so it's useful for humans too. |
| [`audits.md`](audits.md) | Review prompts for pull requests and milestone audits. |
| [`docs/logic/`](docs/logic/) | A flowchart and logic note for each feature's business logic. |

## Repository layout

| Path | Contents |
|------|----------|
| `backend/` | Django project, one app per area (see Stack Mapping in `architecture.md`) |
| `frontend/` | Flutter app |
| `docs/` | Spec and logic notes |

## Getting started

Setup, run, and test commands are in [AGENTS.md](AGENTS.md#setup-and-commands) and will be filled in as the backend and frontend are scaffolded. For local settings, copy `.env.example` to `.env`. Never commit `.env`.
