# Architecture Guidelines

> **Status: Draft.** Project-specific sections reflect *Dinner Bell: Pilot Application Specification*, draft v4 (Nov 16, 2026). Spec references such as **§13.2** or **[D6]** point to that document. Items marked **Proposed** are interpretations of the spec that the team has not yet confirmed; follow them as the current default until the team confirms or replaces them. Items marked **TODO** wait on open questions (spec §16) and must not be settled silently. When the spec changes or the team learns something new, update this file and record the change in the [Decision Log](#decision-log).

This document defines how code in this project is structured and what every change must preserve. It is written for both human engineers and AI coding agents. **Read it before writing or reviewing code.** Follow it by default, and deviate only as described in [Applying These Rules](#applying-these-rules).

The general principles are stack-agnostic. Where they name a layer (handler, service, DAO), use the [Stack Mapping](#stack-mapping) to find the equivalent construct in this codebase.

**Related files**
- `AGENTS.md`: directs coding agents to read this file.
- `audits.md`: review prompts for checking code against these principles. Use the diff-scoped prompts for pull-request reviews and the full-application prompts at milestones. These are review tools; they are not needed while implementing a change.

> **Abstractions are not the goal.** The patterns below describe responsibilities, not a mandate to add layers, interfaces, or files where they don't solve a real problem. Prefer the simplest structure that keeps each responsibility in its proper place.

---

## Quick Reference: The Eight Rules

| # | Principle | Checkable rule |
|---|-----------|----------------|
| 1 | Preserve clear boundaries | Services own business rules. Handlers translate HTTP input, call services, and map outcomes to HTTP responses. |
| 2 | Design persistence carefully (DAO rule) | Only DAO/repository modules connect to the database or issue queries. Multi-step state changes preserve invariants atomically. |
| 3 | Cohesion and loose coupling | Modules own focused responsibilities and depend on narrow contracts. No unnecessary implementation dependencies or dependency cycles. |
| 4 | Treat external input as unsafe | The server validates every untrusted input before use and rejects invalid values with a safe, defined response. |
| 5 | Protect data and access | Authorization runs on the server. Secrets stay out of source control, queries use parameters, and logs omit sensitive values. |
| 6 | Build testability in | Unit-test business rules with injected doubles; integration-test real boundaries; add a regression test per bug; run the full suite after meaningful changes. |
| 7 | Handle errors and concurrency explicitly | State transitions enforce invariants atomically. Failures keep diagnostic context internally and return safe errors. Relevant race cases have tests. |
| 8 | Make code readable | Related workflow code stays together, state changes are discoverable, and names and comments explain intent. |

---

## Applying These Rules

### Default rules

The eight principles are **strong defaults**. Follow them unless doing so would add complexity without solving a real problem. When that happens:

1. Choose the simplest design that departs from the rules as little as possible.
2. Prefer minor deviations (structure, naming, layering, file organization) over deviations that affect data integrity or security.
3. State every deviation and its reason in your change summary.
4. If the deviation is lasting (a pattern others will follow, or a structural choice), record it in the [Decision Log](#decision-log).

Severity matters more than count. One deviation that risks data integrity is worse than several minor structural ones.

### Non-negotiable rules

These rules protect security and data integrity. Following them never adds enough complexity to justify an exception, so **never deviate from them**. If a task seems to require it, stop and ask a human.

- **Server-side authorization.** Every protected read and write is authorized in backend code, regardless of what the UI shows or hides.
- **Parameterized queries.** Never build a query with string concatenation or interpolation of input values.
- **No secrets in source.** Credentials, API keys, and tokens come from the environment or a secret manager, never from committed files.
- **No sensitive data in responses or logs.** Never return raw exceptions or stack traces to clients, and never log credentials, tokens, or personal data.
- **Server-side input validation.** Untrusted input (requests, files, AI output, webhooks) is validated on the server before it is used or stored.
- **Atomic state changes.** Related writes that must stay consistent commit together or roll back together.
- **Dietary safety (Dinner Bell).** Restriction checks are deterministic code, never AI. A recipe that conflicts with a household restriction never appears without its alert. The app never calls a recipe "allergen-free" or "safe for" anyone. See [Dietary restrictions](#dietary-restrictions-safety-critical).

---

## Layer Model

A request crosses each boundary through one defined responsibility. Each layer calls only the layer directly beneath it.

```
Browser / UI  →  API handler  →  Service  →  DAO / repository  →  Database
```

| Layer | Owns | Must not |
|-------|------|----------|
| **Browser / UI** | Collecting input, presenting results, usability feedback | Be relied on for validation or security |
| **API handler** | Parsing and validating request shape; calling a service; mapping results/errors to HTTP responses | Contain business rules or database access |
| **Service** | Business rules, workflows, permissions, state transitions, transaction scope | Know about HTTP request/response objects; open connections or write queries |
| **DAO / repository** | Connections, queries, ORM operations, persistence mapping | Make business decisions (e.g. lifecycle policy) |
| **Database** | Constrained application state; integrity constraints as a backstop | Be the only place a business rule is enforced |

Summary: **HTTP concerns in the handler, business decisions in the service, database mechanics in the DAO.**

---

## Stack Mapping

Source: spec §3 and §13.2. Use the latest patch release in each version series in §3.2, lock exact versions after compatibility checks in CI, and never use prerelease builds.

### Backend: Python 3.13, Django 6.1, Django Ninja 1.x, PostgreSQL 18, Celery 5.6, Redis 8

The spec's six layers (§13.2) map onto the layer model like this:

| Layer model | Spec layer | Django construct | Location (Proposed) | Responsibilities and limits |
|-------------|-----------|------------------|---------------------|------------------------------|
| API handler | API layer | Django Ninja routers and schemas under `/api/v1/` | `backend/<app>/api.py`, `schemas.py` | Authentication, request-shape validation, call one use case, map the result or domain error to a response. No business rules, no ORM calls. |
| Background entry point | Tasks | Celery tasks | `backend/<app>/tasks.py` | Treated like a handler: receive IDs, call a use case, update the `BackgroundJobRecord`. Never reimplement domain logic (§13.2). |
| Service | Use cases | Named operations (`generate_meal_plan`, `check_household_restrictions`, `build_shopping_list`, `estimate_prices`, `import_recipe`) | `backend/<app>/use_cases.py` | Business workflow, authorization, transaction scope (`transaction.atomic()`), coordinating repositories and providers. Receives repositories and providers as keyword-only parameters. |
| Service | Domain logic | Plain Python functions and data classes, including the data classes repositories return | `backend/<app>/domain.py` | Deterministic rules: quantities, units, restriction checks, budget minimum, price totals. No Django, ORM, or network imports. |
| DAO | Persistence | Repository classes wrapping the Django ORM, each with a `typing.Protocol` contract | `backend/<app>/repositories.py` | The only place that issues ORM queries. Every query on household data is scoped by household. Returns data classes or plain values, never model instances. |
| Schema | Persistence | Django models and migrations | `backend/<app>/models.py`, `migrations/` | Fields, relationships, and database constraints. No workflow logic in models, managers, or signals. |
| External boundary | Provider interfaces and adapters | `typing.Protocol` interfaces with fixture-first adapters | `backend/<app>/providers.py`, `backend/<app>/adapters/` | Only adapters call outside services (store APIs, AI, notifications). Use cases depend on the interface, never the adapter. |
| Database | | PostgreSQL 18 | | Constraints backstop integrity: NOT NULL, UNIQUE, foreign keys, CHECK. |

The spec says "Django ORM owns records" without naming a repository layer. These guidelines put ORM queries in `repositories.py` so the DAO rule (Principle 2) holds and use cases can be tested with fakes (Principle 3).

#### Repositories and use cases

- **Repositories are classes.** Each app's `repositories.py` holds a `typing.Protocol` contract (`RecipeRepository`) and its Django implementation (`DjangoRecipeRepository`). The Protocol lists only the methods other code calls.
- **Repositories return data classes or plain values, never model instances.** Returning model instances would let callers run hidden queries through related fields (`plan.planned_meals.all()`). Data classes live in the `domain.py` of the app whose repository returns them. Shape each one for what its callers need instead of copying every model field. Write methods also take data classes or plain values. Converting between models and data classes happens only inside the repository.
- **Use cases receive repositories and providers as keyword-only parameters**, and never import a concrete repository or adapter. API handlers and Celery tasks pass in the real implementations, and tests pass fakes. A use case that calls another app's use case passes along the dependencies that use case needs.
- **Fakes** live in the owning app's `tests/fakes.py` (`FakeRecipeRepository`), so other apps' tests can reuse them.

```python
# recipes/domain.py
@dataclass(frozen=True)
class PlannableRecipe:
    recipe_id: int
    name: str
    servings: int

# recipes/repositories.py
class RecipeRepository(Protocol):
    def list_plannable_recipes(self, household_id: int) -> list[PlannableRecipe]: ...

class DjangoRecipeRepository:
    def list_plannable_recipes(self, household_id: int) -> list[PlannableRecipe]:
        rows = Recipe.objects.filter(...)  # scoped by household
        return [PlannableRecipe(r.id, r.name, r.servings) for r in rows]

# planning/use_cases.py
def generate_meal_plan(household_id: int, *, recipes: RecipeRepository,
                       plans: MealPlanRepository, ranker: RecommendationProvider) -> ...:
    ...

# planning/tasks.py or api.py: the entry point chooses the real implementations
generate_meal_plan(household_id, recipes=DjangoRecipeRepository(),
                   plans=DjangoMealPlanRepository(), ranker=...)
```

**Proposed Django apps** (from the spec §7 data model):

| App | Owns |
|-----|------|
| `accounts` | Account, authentication integration |
| `households` | Household (ZIP code, size, budget, meal slots, preferences) |
| `dietary` | DietaryTag, HouseholdRestriction, RecipeDietaryLabel, LabelReport, the restriction check |
| `recipes` | Meal, Recipe, RecipeIngredient, Ingredient, RecipeSource, recipe import |
| `pricing` | Store, Product, PriceObservation, ingredient-to-product matches, price provider, product matching |
| `planning` | MealPlan, PlannedMeal, plan generation and editing, the recommendation provider and its rankers |
| `shopping` | ShoppingList, ShoppingListItem, unit conversion |
| `feedback` | MealFeedback, FeedbackTicket |
| `evaluation` | EvaluationRating, NDCG metrics, ranker comparison (including the random-order baseline) |
| `jobs` | BackgroundJobRecord, DeviceToken, notification provider |

**Dependency direction (Proposed):** App dependencies form a one-way graph (Principle 3). Apps are grouped into tiers, and an app may import only apps in lower tiers, never apps in its own tier or a higher one.

| Tier | Apps | What they depend on |
|------|------|---------------------|
| 1 | `accounts` | Nothing |
| 2 | `households`, `jobs` | `accounts`: households, job records, and device tokens belong to an account |
| 3 | `recipes` | `households` for user-added recipes; `jobs` for URL imports |
| 4 | `dietary`, `pricing` | `dietary`: `households` for restrictions, `recipes` for labels and label reports. `pricing`: `recipes` to match ingredients to products, `jobs` for price lookups |
| 5 | `planning` | `dietary` for the restriction check, `recipes`, `pricing` for cost estimates, `jobs` for plan generation |
| 6 | `shopping`, `feedback`, `evaluation` | `shopping`: approved plans, restriction alerts, prices. `feedback`: planned meals. `evaluation`: recipes and `planning`'s rankers |

What this order implies:
- Spec §7 lists an optional product match on RecipeIngredient. Because `recipes` sits below `pricing`, the match is stored in `pricing`, not as a field on `RecipeIngredient`.
- Rankers belong to `planning`'s recommendation provider (see [Integration Rules](#integration-rules)). `evaluation` imports them to score them; `planning` never imports `evaluation`.

An app uses another app through that app's use cases or repositories, not by querying its models directly. Foreign keys to a lower tier's models are fine. Don't use Django signals for business rules, because they hide state changes (Principle 8).

**TODO:**
- Decide the mechanism for follow-up work that has to run upward across tiers, such as re-checking approved plans in `planning` when a recipe is edited in `recipes` (§5.2).
- Decide how the learned taste model in `planning` gets the evaluation ratings it trains on (§11.2) without `planning` importing `evaluation`, for example by passing the ratings into the ranker.

### Frontend: Flutter and Dart (web first, phone-sized layouts)

| Layer model | Flutter construct | Location (Proposed) | Responsibilities and limits |
|-------------|-------------------|---------------------|------------------------------|
| UI | Screens and widgets, organized by feature | `frontend/lib/features/<feature>/screens/`, `widgets/` | Presentation and usability-only validation. Designed and tested at phone size first. |
| UI state | State management per feature | `frontend/lib/features/<feature>/state/` | Screen state and calling the API client. No HTTP calls. **TODO:** choose the state-management approach. |
| API client | Typed client for `/api/v1/` | `frontend/lib/core/api/` | The only code that makes HTTP requests. Maps server errors to user-facing messages. |
| Device features | Interfaces for notifications, camera, local storage | `frontend/lib/platform/` | Web implementations now; native implementations swap in for the mobile apps without changing other code (§3.1). |

### Tests

| Level | Backend | Frontend |
|-------|---------|----------|
| Unit | Domain logic with no database. Use cases with fake repositories and fake providers passed in as arguments; these tests run with database access (Django's `TestCase`, or `django_db` in pytest-django) so `transaction.atomic()` works, while the fakes supply all the data. | Widgets and state with a fake API client |
| Integration | Repositories and API endpoints against a real PostgreSQL test database; Celery tasks run eagerly. Test all-or-nothing (rollback) behavior here with real repositories, since a rollback doesn't undo writes to a fake. | API client against a test server |
| User workflow | | `integration_test` flows at phone size |

Location (Proposed): `backend/<app>/tests/`, `frontend/test/`, `frontend/integration_test/`. **TODO:** choose the backend test runner (Django's runner or pytest-django).

---

## Integration Rules

Every tool and external service fits the layer model through one owning layer and a narrow contract. Adapters are fixture-backed first; real APIs and AI providers go behind the same interface (§13.2).

| Tool / service | Purpose | Owning layer | Contract and rules | Status |
|----------------|---------|--------------|--------------------|--------|
| PostgreSQL via Django ORM | Application state | Repositories | Constraints protect integrity. Schema changes only through migrations. | Decided |
| Celery + Redis | Slow work: recipe extraction, price lookups, plan generation (§8) | Tasks call use cases | Pass IDs, not model objects. Idempotent where practical, a bounded number of retries for temporary failures, and a retry never creates a duplicate recipe (§9). Status moves through queued, running, needs review, failed. | Decided |
| Flower | Operator monitoring | Operations only | Restricted to authorized operators; never exposed to app users (§12). | Decided |
| Recipe source adapter | URL import; later photo, screenshot, and social inputs | `recipes` provider | Returns an editable draft that always waits in **needs review**. Prefer structured page data and fall back to the AI provider. Never execute page content. See [URL imports](#recipe-import). | Decided |
| Price provider | Price estimates by ZIP code | `pricing` provider | Store API adapters and a fixture adapter share one interface. Accepts an optional store filter. Every price is stored with source, store or location, currency, and observed time. | Fixture fallback decided [D8a]; APIs TODO (§16 Q4) |
| Product matching provider | Ingredient to store product | `pricing` provider | Unmatched items are flagged, never guessed silently. | TODO (§16 Q8) |
| Recommendation provider | Recipe ranking | `planning` provider | Food.com baseline first, then the learned taste model behind the same interface (§11.2). Ranking never overrides restriction ordering or alerts. | Decided |
| Dietary labeling provider | Automated labeler (future sprint) | `dietary` provider | Writes to the same strict label field with source `labeler` recorded. | Future |
| Hosted AI provider | Extraction fallback; later the labeler and substitution chat | Behind the provider that uses it | Structured outputs validated against a schema. Record prompt and model versions. Send no personal data the task doesn't need, including ZIP codes and restrictions. Confirm retention and training terms before using real pilot data (§10). | TODO (§16 Q10) |
| Notification provider | Browser notifications now, native push later | `jobs` provider | Web and mobile use different implementations of one interface. The in-app job status screen is the dependable path (§9). | TODO (§16 Q5) |
| Authentication provider | Login | `accounts` | Required on every household data endpoint. | TODO (§16 Q10) |
| Object storage, deployment target, observability | Images, hosting, monitoring | TODO | Record each choice here and in the Decision Log once made. Logs carry job IDs and operational events without credentials or unnecessary personal data (§12). | TODO (§16 Q10) |
| Firebase | Mobile test-build distribution only | Release process, not runtime | Don't add Firebase SDKs or services to the app without a recorded decision [D1a]. | Decided (mobile phase) |

---

## Dinner Bell Rules

These rules come from the spec and apply on top of the eight principles.

### Dietary restrictions (safety-critical)

Source: §5, [D6, D6a, D6b]. These rules are non-negotiable.

- The restriction check is deterministic domain code that reads the strict dietary label field. Never use AI for it.
- The check returns **which** restriction each recipe conflicts with, not only pass or fail.
- Conflicting recipes are ranked below fitting ones, not excluded. The user chooses to keep or remove each one.
- A conflicting recipe never appears without an alert naming the conflict ("Contains peanuts", "Not vegetarian"). A kept recipe's alert stays visible on the planned meal and on the shopping list items it adds.
- The check runs again whenever a recipe is edited, an ingredient is swapped, or household restrictions change, including against approved plans.
- Stored labels are authoritative whatever their source. Every label records its source (team, user, or labeler) and when it was added.
- Wording: describe what was checked ("no conflicts with your listed restrictions found"). Never write "allergen-free" or "safe for." Every recipe and shopping list shows a reminder to check product labels.
- Test fixtures include the hard cases in §5.4: hidden allergens (soy sauce, Worcestershire sauce, "natural flavors," store-bought broths) and truncated ingredient names ("peanut oil" read as "oil").

### Deterministic logic and AI

Source: §10. Restriction checks, budget checks, price totals, and standard unit conversion are deterministic code. Ambiguous units ("a handful") are flagged for the user, never guessed. AI is used only behind provider interfaces, and its output is validated before use.

### Budget and pricing

Source: §4.4, §4.5, [D4].

- The $5 per person per meal minimum is a configuration value with its source and as-of date, never hard-coded in logic.
- Over-budget plans are flagged with the amount ("$6 over budget"), never hidden or blocked.
- Totals are labeled as estimates and show price source and retrieval time. If any item has no price, the total says it is incomplete.

### Plans and concurrent edits

Source: §4.1, §4.3, [Q3]. `MealPlan` carries a version number. **Proposed mechanism:** a save succeeds only if the client's version matches the stored version (a conditional update that increments it). Otherwise the API returns `409 Conflict` and the app prompts the user to refresh. Approved plans keep the edits that led to them.

### Recipe import

Source: §4.2, §12.

- **Proposed URL safeguards:** accept only `http` and `https`; reject hosts that resolve to private, loopback, or link-local addresses, including after redirects; set connection timeouts and a maximum response size. These implement the spec's requirement to validate and constrain imported URLs.
- Treat fetched pages as data. Never execute their scripts.
- Imported recipes stay in **needs review** until the user confirms them, and can't be used in a plan before then.
- When extraction fails, show a plain-language message ("We couldn't read this page") with a manual entry option. Never show internal errors.

### Placeholder screens

Source: §13.1, [D2]. Placeholder screens are labeled "coming soon," call no backend logic, and never show sample data as if it were real. No endpoint reports success for work it didn't do.

### Data and privacy

Source: §11.1, §12.

- Allergies and dietary restrictions are sensitive. ZIP codes are personal data: store only what pricing needs and share them only with price lookups.
- Research data (Food.com) is kept separate from real user data. `EvaluationRating` is kept separate from `MealFeedback`.
- Define retention and deletion for accounts and imported content.

### Design seams for future goals

Source: §15. Don't build future features, but don't block them. The seams most likely to matter during implementation:

- Keep `Account` separate from `Household` (member accounts later).
- Keep restrictions in their own table so they can move from household to member.
- Store plan start and end dates, not a week number. Planned meals store dates and slot times.
- Generate shopping lists through one use case that can later subtract pantry stock. Shopping list items have an item type.
- Keep feedback types and the survey model extensible.

See §15 for the full list.

### Documentation for agent-written business logic

Source: §13.3, [Q8]. Any coding agent that writes business logic also adds, in the same change:

- a Mermaid flowchart of the decision paths it implemented, and
- a logic note explaining its choices and assumptions.

**Proposed location:** `docs/logic/<feature>.md`, one file per feature, holding both. The logic note is what spec §13.3 calls a "decision-log entry." `docs/logic/README.md` has the template.

**Where each kind of decision is recorded:**

| Record | Holds | Who updates it |
|--------|-------|----------------|
| Spec §17 decision log | Product decisions (what the app does) | The team, when the spec changes. Agents don't edit the spec unless the task asks them to. |
| [Decision Log](#decision-log) in this file | Architecture decisions and lasting deviations from these rules | Whoever makes the decision, in a reviewed change |
| `docs/logic/<feature>.md` | One feature's flowchart and logic note: the choices and assumptions behind its business logic | Whoever writes or changes that logic, in the same change |

---

## 1. Preserve Clear Boundaries

Each layer has a clear responsibility and communicates through explicit interfaces, so changes stay local. A UI change should not require rewriting queries; a persistence change should not spread through request handlers.

**Do**
- Put every business rule (eligibility, state transitions, workflow ordering, invariants) in the service layer.
- Keep handlers thin: parse → validate shape → call service → map outcome to a response.
- Define each rule once. If two code paths need it, both call the same service function.

**Don't**
- Implement workflow rules in route/view modules.
- Split one rule across a handler and a service.
- Encode lifecycle policy (e.g. "a recipe in needs review can't be used in a plan") in persistence code.

**Agent check:** Before finishing a change, confirm no new business logic was added to a handler or DAO.

---

## 2. Design Persistence Carefully (the DAO Rule)

**Only the DAO/repository layer opens database connections and issues queries** (raw SQL or ORM operations). A single persistence boundary keeps schema and database changes controllable.

**Do**
- Route every read and write through a DAO/repository.
- Let a service coordinate multiple repositories inside one transaction. Repositories do not own or call each other.
- Wrap multi-step state changes in a transaction so related writes commit together or roll back together.
- Use database constraints (NOT NULL, UNIQUE, foreign keys, CHECK) to protect integrity.

**Don't**
- Open connections, create sessions, or run queries/ORM calls in handlers, services, UI code, or utilities.
- Have one repository call another to complete a workflow — that coordination belongs to a service.

**Pattern**
```
API handler  →  build_shopping_list  →  planning repositories ─┐
                (applies rules,         shopping repositories ─┴→  Database
                 owns transaction)                                 (constraints)
```

**Agent check:** Search your diff for connection creation, session use, and query/ORM calls. Every hit must be inside a DAO/repository module.

---

## 3. Design for Cohesion and Loose Coupling

Keep related behavior together and depend on narrow, explicit interfaces. Cohesion and coupling are **separate** qualities — a focused module can still be tightly coupled. Assess both.

**Cohesion — do**
- Group code by feature, resource, or layer so related rules live together (e.g. all meal plan generate, edit, and approve rules in `planning/use_cases.py`).
- Split a module when it combines work that changes for *different reasons*.
- Remember: file size alone is not evidence of weak cohesion.

**Coupling — do**
- Have services depend on a small repository contract (interface/protocol), not on a concrete database implementation.
- Let application setup choose the implementation: real adapter for the running app, a fake for tests.
- Isolate external services (AI/LLM APIs, webhooks, third-party APIs) behind their own module.

**Don't**
- Let routes or workflows depend directly on ORM sessions or models when a repository exists.
- Leak concrete database details through persistence seams.
- Introduce dependency cycles between modules.

**Pattern**
```
generate_meal_plan ──uses──▶ recipe repository (contract)
                               ▲                 ▲
                     implements│                 │implements
                        ORM repository    Fake repository
                         (production)         (tests)
```

---

## 4. Treat External Input as Unsafe

Validate untrusted input at the server boundary before it becomes application state. Requests, uploaded files, AI/LLM output, webhooks, and third-party responses may be malformed, oversized, stale, or hostile. Browser checks improve usability but protect nothing.

**Validation by layer**

| Layer | Responsibility | Safeguards |
|-------|----------------|------------|
| Browser / UI | Usability feedback only | Check required fields early; never treat as security |
| Handler / API | Request shape and limits | Validate types, required fields, format, length, size, allowed values; reject malformed input |
| Service / domain | Meaning, permission, business rules | Check ownership, allowed state transitions, business invariants |
| DAO / database | Persistence and integrity | Parameterized queries; constraints backstop integrity (not a substitute for business rules) |

**Do**
- Use schemas that reject blank strings, bound lengths, and restrict enumerated values — not just type-check.
- Validate AI/LLM output before using it; prefer structured output parsed against a schema over free text.
- Return a safe, defined error response for invalid input.

**Don't**
- Assume a well-typed request is a valid operation.
- Defer shape checks to individual routes inconsistently.
- Trust client-supplied IDs for ownership without checking on the server.

---

## 5. Protect Data and Access

A working feature can still expose data or credentials. Security decisions belong at a trusted server boundary and must follow the project's data requirements.

**Do**
1. **Define the access model:** what data needs protection, and who may read or change it.
2. **Enforce access on the server:** reject unauthorized reads and writes in backend code, regardless of what the UI hides.
3. **Protect secrets and logs:** supply credentials at deployment (environment/secret manager); keep tokens, passwords, and personal data out of logs and error messages.
4. Always use parameterized queries.

**Don't**
- Commit secrets, API keys, or credentials to source control (including example configs with real values).
- Rely on hidden buttons or client-side routing as access control.
- Log request bodies, headers, or objects that may contain credentials or personal data.
- Build queries with string concatenation or interpolation.

---

## 6. Build Testability In

Make important behavior testable at the unit and system boundaries where it may fail. Coverage shows untested paths, not correctness — tests must check behavior that users and other components rely on, including failure cases.

**Match the test level to the behavior**

| Level | Scope | Checks |
|-------|-------|--------|
| Unit | Business rule or UI component with injected fakes | Rules in isolation |
| Integration | Real database, network, serialization | Components work together |
| User workflow | Browser → API → stored result | Complete user paths |

**Do**
- For each important workflow, cover: normal, boundary, invalid, unauthorized (where applicable), duplicate, missing-resource, and failure cases.
- Inject test doubles through contracts rather than patching globals.
- Parameterize repeated cases.
- Add a regression test for every bug fixed.
- Use coverage to find gaps, not as a target.
- Run the complete suite after meaningful changes; keep it running in CI.

**Don't**
- Treat a passing API test as proof the whole user path works.
- Rely on brittle global patches/monkeypatching where dependency injection is possible.

**Agent check:** Every behavior change ships with tests that would fail without the change.

---

## 7. Handle Errors and Concurrency Explicitly

Define what happens when an operation fails or two requests change the same state. Silent failures and racing updates leave data inconsistent; explicit state transitions make outcomes visible and testable.

**For every state-changing operation, define three guarantees:**

| Guarantee | Requirement | Outcome |
|-----------|-------------|---------|
| Atomic updates | Commit related writes or roll them all back | Core writes commit together |
| Safe failures | Return a stable error; log the internal cause | No raw exceptions in responses |
| Concurrent & repeated requests | Lock or claim state; define retry/idempotency behavior | No lost updates or duplicate effects |

**Do**
- Identify the invariants each operation must preserve, and enforce them inside the transaction.
- Log exceptions privately with diagnostic context; return a stable client message (e.g. "Database unavailable", not the driver's exception text).
- When steps are intentionally separate (e.g. a secondary step whose failure should not undo the primary one), make the response reflect what was actually stored — never report failure for something that committed.
- Test injected failures, duplicate retries, and simultaneous requests. Sequential tests verify conflict handling; only simultaneous tests verify concurrency safety.

**Don't**
- Swallow exceptions (`except: pass`, empty catch blocks).
- Return stack traces or raw exception text to clients, including from health checks.
- Assume requests arrive one at a time.

---

## 8. Make Code Readable

Use names and structure that reveal intent to the next engineer reviewing the code. Humans must be able to verify AI-generated changes rather than rely on memory or guesswork.

**Do**
- Keep each workflow easy to locate: a change to one workflow should lead a reviewer to a small set of related functions.
- Give state-changing operations explicit names (`approve_meal_plan`, not a generic `update_meal_plan` that can also approve).
- Keep each function focused on a comprehensible step; a function may coordinate several steps if it has one clear purpose.
- Write comments that explain *non-obvious decisions*, not what the code already says.
- Split code when it clarifies responsibility — not to hit a line-count target.

**Don't**
- Build a single root component/module that holds every view's state, fetches, mutations, and dialogs.
- Hide state changes inside generically named functions.

**Example**
```python
# Less clear: the reviewer can't tell what this does or what it changes
def process(data):
    validate(data)
    apply_changes(data)
    save(data)

# Clearer: same steps, intent and effects are visible
def confirm_imported_recipe(recipe_id, reviewed_fields):
    ensure_recipe_is_awaiting_review(recipe_id)
    apply_reviewed_fields(recipe_id, reviewed_fields)
    mark_recipe_ready_for_planning(recipe_id)
```

---

## Workflow for Coding Agents

When implementing a change:

1. **Locate the right layer** for each piece of the change using the Layer Model.
2. **Reuse existing services and repositories** before creating new ones; don't duplicate a rule.
3. **Validate inputs** at the handler (shape) and service (meaning/permission).
4. **Wrap multi-step writes** in a transaction owned by the service; define error and concurrency behavior.
5. **Write or update tests** at the appropriate level, including failure and edge cases.
   - For business logic, add or update the flowchart and logic note in `docs/logic/` ([Dinner Bell Rules](#documentation-for-agent-written-business-logic)).
6. **Run the full test suite** and fix failures before reporting completion.
7. **Self-review the diff** against the Pre-Merge Checklist below.
8. **Report deviations**: state any departure from these rules and why (see [Applying These Rules](#applying-these-rules)). Add lasting decisions to the [Decision Log](#decision-log).

When **auditing** (not changing) code, use the prompts in `audits.md`.

---

## Pre-Merge Checklist

- [ ] No business rules in handlers, UI, or DAOs
- [ ] No database connections, sessions, or queries outside DAO/repository modules
- [ ] Multi-step state changes are transactional and preserve invariants
- [ ] Services depend on contracts, not concrete persistence; no new dependency cycles
- [ ] All untrusted input (requests, files, AI output, webhooks) validated server-side
- [ ] Authorization enforced on the server for every protected read and write
- [ ] No secrets in source; no sensitive values in logs or error responses; queries parameterized
- [ ] Errors are logged internally and returned as stable, safe messages; no swallowed exceptions
- [ ] Concurrent/repeated requests can't cause lost updates or duplicate effects
- [ ] Tests cover normal, invalid, failure, and (where relevant) unauthorized and concurrent cases
- [ ] Regression test added for any bug fix; full suite passes
- [ ] Names reveal intent; state changes are discoverable; comments explain non-obvious decisions
- [ ] No Non-negotiable rule was broken
- [ ] Every deviation is stated in the change summary; lasting ones are in the Decision Log

**Dinner Bell additions**
- [ ] Restriction checks stay deterministic, name the conflict, and rerun on every trigger in §5.2
- [ ] No "allergen-free" or "safe for" wording; label reminders present where required
- [ ] No ORM queries outside `repositories.py`; household data queries scoped by household; repositories return data classes or plain values, never model instances
- [ ] Use cases receive repositories and providers as parameters; tests pass fakes instead of patching
- [ ] Celery tasks only call use cases; retries can't create duplicates
- [ ] AI calls send only the data the task needs and validate structured output
- [ ] Placeholder screens call no backend logic and show no fake data
- [ ] Business logic changes include a flowchart and logic note in `docs/logic/`

---

## Decision Log

Record lasting design decisions and accepted deviations here so humans and agents can build on them instead of relitigating them. Add newest entries at the top. When a decision replaces an earlier one, mark the old entry *Superseded* and link the new one rather than deleting it.

**Entry format**

```
### YYYY-MM-DD: Short title
- Decision: what was decided
- Context: the problem or new insight that prompted it
- Rule affected: which principle it follows, refines, or deviates from (if any)
- Consequences: what this means for future code
```

### 2026-11-18: Repository classes, injected dependencies, and data classes
- Decision: Each app's repository is a class with a `typing.Protocol` contract in the same `repositories.py`. Use cases receive repositories and providers as keyword-only parameters; API handlers and Celery tasks pass the real implementations, and tests pass fakes. Repositories return data classes or plain values, never model instances. Use-case tests run with database access so `transaction.atomic()` works, and rollback behavior is tested in integration tests with real repositories.
- Context: If use cases imported repository functions directly, tests could swap in fakes only by patching, which Principle 6 discourages. `transaction.atomic()` opens a database connection, so use-case tests need the test database even with fakes. Returning model instances would let use cases and domain code run hidden queries through related fields, breaking the DAO rule, and would keep the restriction check from being pure domain code.
- Rule affected: Principles 2, 3, and 6.
- Consequences: Every repository has a Protocol, a Django implementation, and a fake in `tests/fakes.py`. Repositories contain the code that converts models to data classes. Domain logic, including the restriction check, takes plain data. See [Repositories and use cases](#repositories-and-use-cases).

### 2026-11-18: App dependency tiers (Proposed)
- Decision: Group the Django apps into six tiers; an app imports only apps in lower tiers. Ingredient-to-product matches live in `pricing`. Rankers live in `planning`'s recommendation provider, and `evaluation` imports them to score them.
- Context: The earlier rule split apps into lower and higher levels but left out `jobs` and didn't order apps within each level, so `dietary`, `recipes`, and `pricing` could have ended up depending on each other. The apps table also gave rankers to `evaluation`, while Integration Rules gave the recommendation provider to `planning`.
- Rule affected: Principle 3.
- Consequences: Work that has to run upward across tiers (the §5.2 re-checks) and the taste model's training data still need a mechanism; see the TODOs under Dependency direction. **Needs team confirmation.**

### 2026-11-17: Adopt the pilot spec's layering, with repository modules as the DAO
- Decision: Map spec §13.2 onto the layer model: Ninja routers as handlers, use cases plus pure domain logic as the service layer, `repositories.py` as the only place for ORM queries, and providers with adapters as the external boundary. Celery tasks are entry points that call use cases.
- Context: The spec says "Django ORM owns records" but doesn't say where queries live. Putting them in repositories keeps the DAO rule and lets use cases be tested with fakes.
- Rule affected: Principles 1, 2, 3.
- Consequences: Models hold fields and constraints only. Confirmed 2026-11-18; the repository classes entry above sets how repositories are written and used.

### 2026-11-17: Optimistic version check for plan edits (Proposed)
- Decision: `MealPlan.version` must match on save, or the API returns 409 and the app prompts a refresh.
- Context: Spec requires a refresh prompt when a plan changed on another device [Q3].
- Rule affected: Principle 7.
- Consequences: Plan-editing endpoints accept the client's version. Tests cover simultaneous edits. **Needs team confirmation.**

### 2026-11-17: Dietary restriction checks are deterministic and alert-based (spec D6, D6a, D6b)
- Decision: Conflicting recipes are ranked lower and alerted, not excluded. Checks are deterministic, read one strict label field, and record label source.
- Context: Others in the household may still eat a conflicting meal; restrictions are safety-critical.
- Rule affected: Added to Non-negotiable rules.
- Consequences: See [Dietary restrictions](#dietary-restrictions-safety-critical).

### 2026-11-17: Fixture-first providers (spec D8, D8a, §13.2)
- Decision: Every external dependency sits behind a provider interface with a fixture-backed adapter built first. Fixture prices are the fallback if store API access doesn't come through.
- Context: Store API access for a student pilot is uncertain, and fixtures keep the pilot usable and testable.
- Rule affected: Principles 3 and 6.
- Consequences: Real adapters are added without changing use cases.

### 2026-11-17: Web first in Flutter, device features behind interfaces (spec D1, D1a)
- Decision: Build a Flutter web app at phone size first. Notifications, camera, and local storage go behind interfaces in `lib/platform/`. Firebase is used only for mobile test-build distribution.
- Context: Mobile is the long-term focus, and the web app must carry over without a rewrite.
- Rule affected: Principle 3.
- Consequences: Native implementations replace web ones in the mobile phase.

### 2026-11-17: Agents document business logic (spec Q8, §13.3)
- Decision: Agent-written business logic ships with a Mermaid flowchart and a logic note, proposed location `docs/logic/<feature>.md`.
- Context: The team needs to check agent logic against the spec.
- Rule affected: Principle 8.
- Consequences: Part of the Pre-Merge Checklist.

### 2026-11-17: Rules are strong defaults with a small non-negotiable set
- Decision: The eight principles are defaults, not absolute requirements. A short list of security and data-integrity rules is non-negotiable.
- Context: Following every rule exactly can add complexity without solving a real problem, and the spec is not yet final.
- Rule affected: All; see Applying These Rules.
- Consequences: Agents may deviate from a default rule when it adds unjustified complexity, but must choose the smallest deviation and state it. Non-negotiable rules are never relaxed without human approval.
