# Dinner Bell: Pilot Application Specification

**Status:** Draft v4 for team review
**Last updated:** November 16, 2026
**Audience:** The Dinner Bell team, instructors, and AI coding assistants
**Replaces:** Budget Meal Planning Assistant, Application Specification (working draft, Nov 9)

This version folds in the team decisions recorded in *Dinner Bell: Spec Disagreements to Decide* (D0 to D8 and the quick-decision tables), follow-up decisions made after that review, Bella's notes, and the pilot spec feedback doc. Decision references like **[D4]** point to the decision log in §17. Anything not yet settled is marked **Open** and collected in §16. Future goals are kept in this document on purpose: the pilot should be built so they can be added without redesign (§15).

Key dates: the report draft is due next week, and the final report PDF is due Saturday, December 12, 2026 at 11:59 p.m. This is a two-semester project.

### Changes in v4

- **Platform [D1]:** web first. The team builds a working web app with Flutter and Dart, then moves to mobile apps and app store releases once the core functionality works (§3.1).

### Changes in v3

- **Allergens and restrictions:** recipes that conflict with a household restriction are ranked lower and clearly alerted, and the user chooses to keep or remove them (§5).
- **Dietary labels:** a strict, queryable field with a fixed vocabulary; hand-labeled for the pilot, with an automated labeler planned for a later sprint (§5, §15).
- **Budget:** $5 per person per meal is the minimum budget a user can set (§4.4).
- **Pricing:** fixture prices are the confirmed fallback if store APIs don't come through (§4.5).
- **Firebase:** used only to distribute test builds before the app is store-ready (§3.1).
- **Evaluation:** taste-model method and target defined (§11.2).

---

## 1. Purpose and problem statement

**Main problem [D0]:** People want to cook at home within a budget, but turning recipes into a realistic weekly plan takes time and effort. Dinner Bell curates recipes into a meal plan for the user, fitted to their household's dietary restrictions, preferences, and weekly budget, and turns the approved plan into a price-aware shopping list.

Everything in the pilot is judged by how well it serves that problem. Pantry tracking, waste reduction, and ordering are valuable, but they are future goals rather than what the pilot is measured on.

**Product promise:** Plan it · Shop it · Cook it.

**What success looks like for the pilot:**

- Users accept curated plans with few replacements or edits.
- The recommender reduces the ranking error of a Food.com baseline by at least 30% (§11.2).
- Users understand the plan's estimated cost and are told clearly when it's over budget.
- Every planned recipe that conflicts with a household restriction carries a clear alert; none appears without one.

## 2. Pilot scope at a glance

| Area | In the pilot | Placeholder screen | Future goal |
| --- | --- | --- | --- |
| Accounts | One household account [D7] | | Individual member accounts with invites |
| Household settings | ZIP code, household size, weekly budget, dietary restrictions | | Member profiles, ages, calorie targets, preferred stores |
| Recipes | Curated, hand-labeled recipe catalog; save from website URL; manual entry; meal variants | | Photo, screenshot, and social media inputs; automated dietary labeler |
| Planning | Weekly plan with chosen meal slots; accept, replace, or edit meals [D5] | | Any date range; skip reasons |
| Restrictions | Conflicting recipes ranked lower and alerted; user keeps or removes [D6] | | Allergy tiers; alternatives for one household member |
| Budget | Weekly budget with a $5 per person per meal minimum; clear over-budget notice [D4] | | Local cost reference, strict mode, budget-building help |
| Shopping list | Built straight from the plan, with ZIP-based price estimates and a fixture fallback [D3, D8] | | Pantry subtraction, non-food section, preferred stores, AI substitution chat |
| Pantry | | Yes, labeled "coming soon" | Manual and barcode tracking, consumption updates |
| Feedback | Post-meal rating and changes made; dietary label problem reports; feature requests and issue reports | | Skip reasons, actual amounts used |
| Platform | Flutter web app, designed for phone screens [D1] | | iOS and Android apps, then App Store and Google Play releases |
| Notifications | In-app job status, plus browser notifications where supported | | Native push notifications in the mobile apps, reminders, calendar integration |
| Calendar | | Optional | .ics export, then direct calendar connections |

## 3. Platforms and technology

### 3.1 Client platforms [D1]

- **Frontend:** Flutter and Dart, one codebase that can target web, iOS, and Android.
- **Priority: web first.** The pilot is a Flutter web app. The goal is working functionality first; mobile apps and app store releases follow once the core features work. Mobile remains the long-term focus, so the web app is built to carry over without a rewrite.
- **Pilot distribution:** pilot users open the web app through a link on a phone or computer. No app store accounts or test-build distribution are needed for the pilot.
- **Designing for the move to mobile:**
  - Design and test every screen at phone size first (in phone browsers and phone-sized browser windows), not just on a laptop.
  - Keep device-specific features (notifications, camera access for later barcode scanning, local storage) behind small interfaces, so the mobile apps can swap in native versions without changing the rest of the app.
- **Web limitations we accept for the pilot:** Flutter web has a larger first load and behaves less like a typical website (text selection, screen readers). Browser notifications on iPhones only work when the user adds the site to their home screen, so the in-app job status screen is the dependable way to see results (§9).
- **Mobile phase (later):** Firebase is used only to distribute mobile test builds before the apps are store-ready. iPhone builds require the Apple Developer Program ($99 per year, required for both TestFlight and Firebase iOS test builds) and a Mac with Xcode. Android test builds can be distributed through Firebase without a Google Play account. Public store releases come after that. See *Dinner Bell: Web First or Mobile First* for details.

### 3.2 Backend (selected in the Nov 9 spec, unchanged)

| Component | Version series |
| --- | --- |
| Python | 3.13.x |
| Django | 6.1.x |
| Django Ninja | Latest stable 1.x |
| PostgreSQL | 18.x |
| Celery | 5.6.x |
| Redis (broker) | 8.x |
| Flower (operator monitoring) | 2.2.x |

Use the latest patch release in each series and lock exact versions after compatibility checks in CI. Never use development or prerelease builds for the pilot.

### 3.3 Still to select

Authentication provider, push notification service, deployment target, file and object storage, hosted AI provider and model (with data-processing terms), observability stack, and which store price APIs to use. See §16.

## 4. Users and primary workflows

### 4.1 Household account [D7]

One account represents one household. The household stores its ZIP code, household size, weekly grocery budget, planning schedule (which meal slots to plan), dietary restrictions and allergies, and food preferences. Restrictions are set at the household level for the pilot; individual member accounts and member-level profiles are future goals (§15).

If the same plan is open on two devices and one changes it, the other device prompts the user to refresh before saving, so one edit can't silently overwrite another.

### 4.2 Recipes

**Curated catalog.** The pilot starts from a hand-labeled recipe catalog. Each recipe carries dietary labels (allergens present, and diet labels such as vegetarian) stored in the strict dietary label field described in §5 [D6].

**Meals and variants.** A **Meal** is the parent record (for example, "Chicken tacos"). Each **Recipe** is a specific version of that meal with a foreign key to its parent. Users can edit a recipe or save a variant under the same meal, keeping both versions. Recipes can be tagged and archived.

**Saving from a website.** A user submits a recipe URL. A background job extracts the title, ingredients with quantities and units, instructions, servings, source, and an optional image. The recipe waits in a **needs review** state until the user checks and corrects it. Website parsing sits behind a source adapter so other inputs can be added later.

**When extraction fails.** The user sees a plain-language message (for example, "We couldn't read this page") with the option to enter the recipe manually. No stack traces or internal errors are shown.

**Manual entry** is always available.

### 4.3 Weekly planning [D5]

1. The user chooses which meal slots to plan (for example, dinners only, or all meals). A plan covers one week.
2. Dinner Bell proposes recipes based on dietary restrictions, preferences, budget, ingredient overlap, and variety. Recipes that conflict with a household restriction are ranked lower and alerted (§5).
3. The user can accept, reject, replace, or edit any proposed meal. Users can always override the plan, including keeping a recipe that conflicts with a restriction or a plan that's over budget.
4. The system stores the approved plan and the edits that led to it, since those edits are useful feedback.

### 4.4 Budget [D4, D5]

- The budget is **weekly** and set by the user.
- **$5 per person per meal is the minimum budget a user can set.** For a planned week, the minimum is $5 × household size × number of planned meal slots. For example, a household of 4 planning 7 dinners can't set a weekly budget below $140. The settings screen explains the minimum when a lower amount is entered.
- Store the $5 figure as a configuration value with an as-of date, not hard-coded in logic, so it can be reviewed each semester.
- The planner tries to build plans that stay within the budget.
- If the plan it builds goes over budget, the app notifies the user clearly, with the amount (for example, "$6 over budget"). Over-budget plans are flagged, not hidden or blocked, and the user decides whether to accept.
- Price totals are estimates. Show the price source and when it was retrieved. If some items have no price, say the total is incomplete.

### 4.5 Shopping list and pricing [D3, D8]

The shopping list is built straight from the approved plan. There is no pantry subtraction in the pilot. The user can edit quantities, remove items, and check items off.

Users enter their ZIP code, and Dinner Bell gathers price estimates from store price APIs for that area. Each price is stored with its source, store or location, currency, and the time it was observed. Prices that can't be matched to a product are flagged on the list.

**Fallback (decided):** store APIs may not grant access to a student pilot (the Nov 9 spec found that Walmart's grocery API requires approval into a partner program). If API access doesn't come through, the price provider uses fixture prices (hand-collected and labeled as estimates), so the pilot stays usable either way. Both sources sit behind the same price provider interface.

Unit conversion for standard units is deterministic code. Ambiguous units ("1 stick of butter," "a handful") are flagged for the user to confirm rather than guessed.

**Open:** whether the pilot list sums the same ingredient across meals (§16).

### 4.6 Feedback

- **Post-meal survey:** after a planned meal, ask how much the household liked it and what changes they made.
- **Dietary label problem reports:** every recipe has a way to report a missing or wrong allergen or diet label. Reports go to the team for review and correction [D6].
- **Feature requests and issue reports:** a simple in-app form, since the pilot exists to learn what users want.

Evaluation ratings for the recommender are collected separately (§11.2).

## 5. Dietary restrictions and allergens [D6]

Households can include people with different needs. If one person is allergic to something or is vegetarian, the rest of the household may still eat meals with that ingredient while the person who needs the accommodation eats something else. The pilot therefore **prioritizes** recipes that fit every restriction and **alerts** the user about any that don't, rather than excluding them outright.

### 5.1 Strict dietary label field

- Dietary restrictions and recipe dietary labels are stored as a **strict, queryable field**, not free text: a fixed vocabulary of allergens and diet labels in its own table, linked to recipes and households through join tables.
- However a label gets there (team hand-labeling in the pilot, user corrections, or the future automated labeler), once it is stored it is authoritative: planning, alerts, and queries all read the same field.
- Each label records its source (team, user, or labeler) and when it was added, so labels can be audited and the labeler's output compared with hand labels later.
- For the pilot, catalog recipes are hand-labeled by the team. A sprint later in the project will develop an automated dietary restriction labeler (§15).

### 5.2 Planning behavior

1. The recommender ranks recipes that fit all household restrictions above recipes that conflict with any of them.
2. If a proposed recipe conflicts with a restriction, the plan shows a clear alert naming the allergen or diet rule (for example, "Contains peanuts" or "Not vegetarian").
3. The user chooses to **keep** or **remove** the recipe. Keeping it is a deliberate choice; the alert stays visible on the planned meal and on the shopping list items it adds.
4. The restriction check is deterministic application code, not AI. It runs again whenever a recipe is edited, an ingredient is swapped, or the household's restrictions change, including against plans already approved, and new conflicts are alerted the same way.
5. A conflicting recipe never appears without its alert.

### 5.3 Wording and label reminders

- Every recipe and shopping list shows a short reminder to check product labels, since "may contain" warnings and brand differences only appear on the package.
- The app describes what it checked: "contains ingredients that conflict with your household's restrictions," or "no conflicts with your listed restrictions found." It never says a recipe is "allergen-free" or "safe for" anyone.

### 5.4 Hard cases

Known hard cases for hand-labeling, the future labeler, and test fixtures: ingredients that hide allergens (soy sauce usually contains wheat; Worcestershire sauce contains anchovy; "natural flavors"; store-bought broths and sauces), and extraction errors that drop part of an ingredient name ("peanut oil" read as "oil").

**Open:** how recipes users add themselves get labels in the pilot, and whether the app suggests an alternative for the person who needs the accommodation (§16).

## 6. Functional requirements

**Pilot (must have):**

1. The product shall provide a Flutter web app, designed for phone-sized screens, from a codebase that can later build the iOS and Android apps.
2. The system shall let a household manage ZIP code, household size, weekly budget, meal slots, dietary restrictions, allergies, and preferences.
3. The system shall reject weekly budgets below $5 per person per planned meal and explain the minimum.
4. The system shall provide a curated, hand-labeled recipe catalog.
5. The system shall store dietary restrictions and recipe dietary labels in a strict, queryable field with a fixed vocabulary and a recorded label source.
6. The system shall let users create, view, edit, tag, and archive recipes, and save variants under a parent meal.
7. The system shall accept a recipe URL, create a background extraction job, and show its status.
8. The system shall let users review and correct extracted fields before a recipe is saved or used in a plan.
9. The system shall generate a weekly plan for the selected meal slots that ranks recipes fitting all household restrictions above those that conflict.
10. The system shall alert the user to every planned recipe that conflicts with a household restriction, naming the conflict, and let the user keep or remove it.
11. The system shall let users accept, reject, replace, or edit planned meals and retain the approved version and edit history.
12. The system shall try to keep plans within the weekly budget and clearly notify the user, with the amount, when a plan goes over.
13. The system shall build an editable shopping list from the approved plan.
14. The system shall estimate prices by ZIP code through a price provider, fall back to fixture prices when API data isn't available, show source and freshness, and flag unmatched items.
15. The system shall convert standard units deterministically and flag ambiguous units for user confirmation.
16. The system shall record post-meal ratings and changes made.
17. The system shall let users report dietary label problems, request features, and report issues.
18. The system shall show current background job status in the app, and send browser notifications when jobs finish or fail where the browser supports them.
19. The system shall prompt for a refresh when a plan has changed on another device.
20. The system shall show placeholder screens for planned features, clearly labeled as coming soon.
21. Operators shall monitor Celery workers and tasks through Flower, restricted to authorized operators.

**Future (design for, don't build yet):** see §15.

## 7. Data model

Relationships and exact fields are confirmed during schema design. Entities marked **Future** should not be built in the pilot, but pilot tables shouldn't block them.

| Entity | Pilot | Notes |
| --- | --- | --- |
| Account | Yes | Login identity. Kept separate from Household so several accounts can join one household later. |
| Household | Yes | ZIP code, household size, weekly budget, meal slots, preferences. |
| DietaryTag | Yes | Fixed vocabulary of allergens and diet labels (for example, peanut, wheat, vegetarian), with a type (allergen or diet). |
| HouseholdRestriction | Yes | Household, DietaryTag. Future: attach to a member instead of the household. |
| RecipeDietaryLabel | Yes | Recipe, DietaryTag, label source (team, user, or labeler), and when it was added. |
| Meal | Yes | Parent record, such as "Chicken tacos." |
| Recipe | Yes | Variant of a Meal (FK). Servings, instructions, source, review status, visibility. |
| RecipeIngredient | Yes | Recipe, ingredient, quantity, unit, preparation note, optional product match. |
| Ingredient | Yes | Normalized identity and aliases. |
| RecipeSource | Yes | Source URL and provenance. |
| MealPlan | Yes | Household, week, budget, status, cost estimate, version (for the refresh check). |
| PlannedMeal | Yes | Plan, date, meal slot, recipe, serving count, any restriction conflicts and the user's keep decision. |
| ShoppingList, ShoppingListItem | Yes | Items from the plan, quantity, estimated price, matched product, checked state. |
| Store, Product, PriceObservation | Yes | Store location, product and package size, price with source (API or fixture), location, currency, and observed time. |
| MealFeedback | Yes | Planned meal, rating, changes made. |
| EvaluationRating | Yes | User, recipe, 0 to 5 rating from the evaluation survey (§11.2), train or held-out split. Kept separate from MealFeedback. |
| LabelReport | Yes | Recipe, reported dietary label problem, status of the team's review. |
| FeedbackTicket | Yes | Feature requests and issue reports. |
| BackgroundJobRecord | Yes | User-facing job status, keyed by job ID. |
| DeviceToken | Yes | Push notification registration per device. |
| HouseholdMember | Future | Member profiles, individual restrictions, optional age and calorie target. |
| HouseholdInvite | Future | Expiring, single-use invite for member accounts. |
| PantryItem, PantryAdjustment | Future | Inventory and auditable consumption changes. |
| PreferredStore | Future | Household's chosen stores. |

## 8. API shape (provisional)

Versioned routes under `/api/v1/`, with Django Ninja schemas, validation, and generated docs. Route names are not final.

- `/users/me`, `/household`, `/household/restrictions`, `/dietary-tags`
- `/meals`, `/recipes`, `/recipes/{id}`, `/recipes/{id}/variants`, `/recipes/{id}/dietary-labels`
- `/recipe-imports` (submit a URL, check status, confirm reviewed content)
- `/meal-plans`, `/meal-plans/{id}` (generate, retrieve, edit, approve; includes restriction alerts and the plan version for the refresh check)
- `/shopping-lists`, `/shopping-lists/{id}`
- `/prices` (estimates by ZIP code through the price provider)
- `/feedback/meals`, `/feedback/label-reports`, `/feedback/tickets`
- `/evaluation/ratings`
- `/devices` (register push tokens)
- `/jobs/{id}` (job status)

Quick reads and writes are synchronous. Recipe extraction, price lookups, and plan generation run through Celery where they may be slow.

## 9. Background jobs and notifications

1. The user submits a request (for example, a recipe URL). The server creates a job record with an ID and responds right away.
2. A Celery worker runs the job. Status moves through queued, running, needs review, or failed.
3. The user can leave the app. When the job finishes or fails, the server sends a browser notification where the browser supports it.
4. When the user opens the app, it shows current status from the job record. This is the dependable path in the web pilot, since some browsers (including Safari on iPhones, unless the site is added to the home screen) won't show notifications.

Native push notifications come with the mobile apps. The notification service is behind an interface so the web and mobile versions can use different implementations; which service to use is still open (§16). Jobs are idempotent where practical, retry a bounded number of times for temporary failures, and never create duplicate recipes on retry.

## 10. What is deterministic and what uses AI

| Task | Approach |
| --- | --- |
| Restriction checks and alerts | Deterministic code reading the strict dietary label field. Never AI. |
| Budget checks and price totals | Deterministic code. |
| Unit conversion | Deterministic for standard units; ambiguous units go to the user. |
| Recipe extraction from a URL | Structured page data when available; hosted AI turns recipe text into an editable draft otherwise. The user always reviews. |
| Recipe ranking | Food.com baseline first; learned taste model evaluated against it (§11.2). |
| Dietary labeling | Hand-labeled by the team for the pilot. The future automated labeler writes into the same strict field, with its source recorded. |

AI runs behind a provider interface with structured outputs and recorded prompt and model versions. Don't send personal data the task doesn't need, including ZIP codes and restrictions, to an AI provider. Confirm the provider's retention and training-use terms before using real pilot data.

## 11. Data and evaluation

### 11.1 Data sources

| Source | Use | Status |
| --- | --- | --- |
| Food.com recipes and ratings | Recipe catalog seed and the recommender baseline | Confirm license and permitted use |
| Team hand labels | Dietary labels for the curated catalog | Decided for the pilot [D6] |
| Pilot user ratings | Ground truth for evaluation (§11.2) | Decided |
| Store price APIs by ZIP code | Price estimates | Access to confirm (§16) |
| Hand-collected prices | Fixture fallback if API access doesn't come through | Decided |

Keep research datasets clearly separated from real user data.

### 11.2 Taste model evaluation (decided)

**Rating collection.** Each pilot user rates a pool of Food.com recipes (roughly 20 to 30; exact size open) from 0 to 5 stars, answering the same question every time, for example "How much would you want this recipe in your plan?" Those ratings are the true relevance scores, since preferences are personal.

**Recipe pool.** Draw each pool as a random sample from the catalog, or a mix across Food.com rating levels, rather than only highly rated recipes. If every recipe in the pool is highly rated on Food.com, the baseline's ordering among them is nearly arbitrary.

**Training and held-out data.** The model is never scored on ratings it learned from. Each user's ratings are split into a training set and a held-out set (or preferences are collected at onboarding and a separate pool is used for evaluation). Both rankers are scored on the same held-out recipes for each user.

**Rankers compared:**

- **Food.com baseline:** ranks the pool by Food.com user ratings, using a minimum review count or a weighted average so a recipe with one 5-star review doesn't outrank a well-reviewed one, with a defined tie-break.
- **Dinner Bell model:** ranks the same pool using what it learned from the user's training ratings.
- **Random order:** reported alongside both, to show how much better than chance each ranker is.

**Metric.** Each ranker orders the whole pool, and NDCG@10 is computed on its top 10:

- DCG@10 = Σ gain(rel_i) / log₂(i + 1) for positions i = 1 to 10 (in 0-indexed code, use log₂(i + 2)).
- IDCG@10 is the same calculation with the user's ratings sorted from highest to lowest.
- NDCG@10 = DCG@10 / IDCG@10.
- **Primary metric:** gain(rel) = 2^rel − 1, which weights getting the top-rated recipes first most heavily (a 5 counts 31, a 4 counts 15).
- **Robustness check:** the same calculation with gain(rel) = rel (the raw rating). Results should hold under both.

**Target.** The Dinner Bell model reduces the Food.com baseline's ranking error, 1 − NDCG@10, by at least 30%. For example, if the baseline scores 0.70, the model must reach at least 0.79. Defining the target as a share of the remaining gap keeps it reachable wherever the baseline lands.

**Reporting.** Average NDCG@10 across users, compare the two rankers user by user, and report a bootstrapped confidence interval rather than a single number, since the pilot will be small.

### 11.3 Other component targets (proposed)

| Component | Target | Status |
| --- | --- | --- |
| Ingredient extraction | Ingredient F1 ≥ 0.85 | Proposed |
| Product matching | Correct product match ≥ 90% | Proposed |
| Meal planner | Plans within budget ≥ 95% | Proposed |

Each learned component is compared with a simple baseline.

## 12. Security, privacy, and reliability

- Require authentication and authorization for all household data.
- Validate and constrain imported URLs. Never execute source content.
- Treat allergies and dietary restrictions as sensitive and safety-critical.
- Treat ZIP codes as personal data: store only what pricing needs, and don't share them beyond price lookups.
- Keep evaluation ratings linked to pilot accounts only as long as the evaluation needs, and describe their use to pilot users.
- Minimize stored personal data, and define retention and deletion for accounts and imported content.
- Keep secrets out of source control.
- Restrict Flower, database, and Redis administration to operators.
- Log job IDs and operational events without credentials or unnecessary personal content.
- Use migrations and backups appropriate to the chosen deployment.

## 13. Build approach [D2]

### 13.1 Screens first

The first version focuses on the screens and flows for the primary functionality: household setup, recipes, weekly planning, restriction alerts, budget, and the shopping list. Planned features that aren't built yet get **placeholder screens** so the team and pilot users can see where they'll live.

Placeholder rules:

- Label placeholder screens clearly as "coming soon."
- Never show sample data as if it were real, and never have an endpoint report success for work it didn't do.
- Placeholder screens call no backend logic.

### 13.2 Layering

1. **API layer:** Django Ninja schemas, auth, validation. Endpoints call use cases and hold no business rules.
2. **Use cases:** named operations such as `generate_meal_plan`, `check_household_restrictions`, `build_shopping_list`, `estimate_prices`, `import_recipe`.
3. **Domain logic:** deterministic rules for quantities, units, restriction checks, and price totals, kept free of network and framework dependencies where practical.
4. **Provider interfaces:** recipe extraction, price lookup, product matching, recommendation, dietary labeling, notifications.
5. **Adapters:** fixture-backed first; real APIs and AI providers behind the same interfaces.
6. **Persistence and tasks:** Django ORM owns records; Celery tasks call use cases and never reimplement domain logic.

### 13.3 Requirements for AI coding agents

Any coding agent that writes business logic for this project must also produce:

- a flowchart (for example, a Mermaid diagram in the repo) showing the decision paths it implemented, and
- a decision-log entry explaining the choices it made and any assumptions.

These go in the repo's docs alongside the code, so the team can check the agent's logic against this spec.

## 14. Delivery phases

1. **Design (through Dec 12):** report, problem statement, data model, architecture diagram, screen flows, and placeholder screens.
2. **Foundation:** Flutter web app shell (phone-sized layouts), Django and Ninja project, PostgreSQL schema, household account, authentication, deployment skeleton, API docs.
3. **Primary flow on fixture data:** catalog with hand labels, weekly planning with restriction alerts, budget checks, shopping list with fixture prices.
4. **Recipe import:** URL extraction, review, background jobs, in-app status and browser notifications.
5. **Pricing and recommendation:** ZIP-based price provider with fixture fallback; Food.com baseline ranker, then the learned taste model evaluated against it.
6. **Automated dietary labeler sprint:** build the labeler, compare its output with hand labels, and write into the same strict field.
7. **Pilot:** invited users through a web link, rating collection, evaluation against §11 targets.
8. **Mobile apps:** iOS and Android builds from the same codebase, test distribution through Firebase, native push notifications.
9. **App store releases and other future goals:** see §15.

## 15. Future goals and design seams

These are out of the pilot, but they shape decisions now. The right-hand column is what the pilot must do so the feature can be added later without redesign.

| Future goal | Source | Design seam in the pilot |
| --- | --- | --- |
| Automated dietary restriction labeler | Team decision | Strict dietary label field with a recorded label source; hand labels serve as its evaluation set. |
| Individual member accounts with expiring, single-use invites | Bella, D7 | Keep Account separate from Household. |
| Member profiles (per-member restrictions, optional age and calorie target) | Bella | Keep restrictions in their own table so they can move from household to member. Treat age and calorie data as sensitive and optional; word calorie features so they don't encourage unhealthy behavior. |
| Alternatives for the household member who needs an accommodation | Team discussion | Restriction checks return which restriction each recipe conflicts with, not just pass or fail. |
| Allergy tiers (allergy, intolerance, dislike) | D6 discussion | DietaryTag has a type field that can grow. |
| Toddler and infant food-safety guidance from a cited source | Bella | Keep safety rules as deterministic, versioned rule data. |
| Pantry: manual entry, then barcode scanning, check-off-to-pantry, and confirmed consumption | M2, D3 | Keep shopping-list generation behind a use case that can later subtract pantry stock. |
| Total ingredient quantities across meals before subtracting the pantry | Feedback | Same use case as above. |
| Any date range for planning | Feedback, D5 | Store plan start and end dates, not just a week number. |
| Skip reasons that don't change the taste profile | Feedback | Keep feedback types extensible. |
| Preferred stores | D8 | Price provider accepts an optional store filter. |
| Local cost reference for the budget minimum; strict mode | M1, Bella | Keep the minimum in configuration with its source and date. |
| Combined local price source across several stores | M3 | Price provider interface supports multiple adapters. Check each source's terms of use; scraping retail sites often violates them. |
| Photo, screenshot, and social media recipe input | Bella, Feedback | Recipe source adapter interface. |
| Shopping-list AI chat for substitutions | Bella | Any substitution reruns the deterministic restriction check. |
| Non-food items on the list, excluded from models | Bella | Shopping-list items have an item type. |
| Calendar export (.ics), then direct calendar connections | Bella | Planned meals store date and slot times. |
| Actual amounts used in the post-meal survey | Bella | Survey model can take more fields. |
| Learned plan optimization and live checkout or ordering | Nov 9 spec | Provider interfaces for planning and stores. |
| iOS and Android apps, test distribution through Firebase | D1 | Same Flutter codebase; phone-sized layouts; device-specific features behind interfaces. |
| Public App Store and Google Play releases | Feedback, D1 | Apple Developer Program ($99 per year) and Google Play registration ($25 one time) when the mobile apps are ready. |

## 16. Open questions

1. **Apple Developer Program (mobile phase):** who pays the $99 per year, or does the team ask the university about a fee waiver?
2. **Labels for user-added recipes:** recipes from a URL or manual entry aren't hand-labeled by the team. Options: the user labels them during review, a deterministic ingredient-keyword check suggests labels, the team labels them, or a combination. Until labeled, how should they appear in plans?
3. **Alternatives for one person:** should the app suggest an alternative dish for the household member who needs the accommodation? If so, restrictions need to record who they apply to (a name or headcount), even before full member profiles.
4. **Store price APIs:** which APIs, and has access been confirmed? The fixture fallback covers the pilot either way.
5. **Notification service:** which service sends browser notifications now and native push notifications later, given Firebase is used only for test-build distribution.
6. **Shopping list totals:** should the pilot list sum the same ingredient across meals (no pantry needed), or list each meal's ingredients separately?
7. **Evaluation details:** exact pool size per user; how training and held-out ratings are split; the Food.com minimum review count or weighting; whether the Food.com license permits this use.
8. **Product matching:** hand-maintained ingredient-to-product mapping for the pilot, or something else?
9. **Food assistance resources:** should the budget screen link to food assistance resources?
10. **Remaining stack choices:** authentication provider, deployment target, object storage, AI provider and terms, observability.
11. **Proposed evaluation targets:** which of extraction F1, product matching, and planner budget are commitments?

## 17. Decision log

| ID | Decision | Spec sections |
| --- | --- | --- |
| D0 | Main problem: curate recipes into a meal plan for the user. | §1 |
| D1 | Web first with Flutter and Dart: get a working web app, then move to mobile apps and app store releases. Mobile remains the long-term focus. (Revised Nov 16 from "mobile alongside web.") | §3.1, §14 |
| D1a | Firebase is used only to distribute mobile test builds before the apps are store-ready. | §3.1, §15 |
| D2 | Screens first; primary functionality with placeholders for planned features. | §13 |
| D3 | Shopping lists come straight from the plan; pantry is a later stretch goal. | §4.5, §15 |
| D4 | $5 per person per meal is the minimum budget a user can set; the planner tries to stay within budget and clearly notifies the user when a plan goes over. | §4.4 |
| D5 | Weekly budget by default; any date range later; users can always override plans. | §4.3, §4.4 |
| D6 | Recipes and households have dietary fields; hand-labeled data for the pilot; user reports of label mistakes. | §5, §4.6 |
| D6a | Conflicting recipes are prioritized lower, not excluded; the user is alerted and chooses to keep or remove them, since others in the household may still eat them. | §5.2 |
| D6b | Dietary restrictions and labels are a strict, queryable field; once a label is stored it is authoritative regardless of source. An automated labeler gets its own sprint. | §5.1, §14, §15 |
| D7 | Household accounts now; member accounts later. | §4.1, §15 |
| D8 | Users enter a ZIP code; pricing from store APIs; preferred stores later. | §4.5 |
| D8a | Fixture prices are the fallback if store API access doesn't come through. | §4.5, §11.1 |
| Q1 | Taste model evaluation: users rate a larger pool; NDCG@10 with 2^rel − 1 gains as the primary metric and raw ratings as a check; target is a 30% reduction in the Food.com baseline's 1 − NDCG@10. | §11.2 |
| Q2 | Post-meal survey: rating and changes made. | §4.6 |
| Q3 | Refresh prompt when a plan changed on another device. | §4.1 |
| Q4 | Deterministic unit conversion. | §4.5, §10 |
| Q5 | Plain-language extraction failure with manual entry option. | §4.2 |
| Q6 | Meals as parent records with recipe variants. | §4.2, §7 |
| Q7 | Feature requests and issue reports in the first version. | §4.6 |
| Q8 | Coding agents produce flowcharts and decision-log entries for business logic. | §13.3 |
| Q9 | Notifications for background job status (browser notifications in the web pilot, native push in the mobile apps). | §9 |
| Later | Aggregation before pantry subtraction, skip reasons, date ranges, non-food section, photo and social input, AI chat, calendar export, toddler guidance, calorie targets and ages. | §15 |

## 18. Sources

From the Nov 9 spec's technology and API review:

- [Django supported versions](https://www.djangoproject.com/download/)
- [Celery documentation](https://docs.celeryq.dev/en/stable/)
- [PostgreSQL version support policy](https://www.postgresql.org/support/versioning/)
- [Redis version management](https://redis.io/docs/latest/operate/oss_and_stack/install/version-mgmt/)
- [Flower documentation](https://flower.readthedocs.io/en/latest/)
- [Walmart OPD grocery API overview](https://www.walmart.io/docs/opd/stores-recipe/grocery-introduction)
- [Walmart Marketplace API access FAQ](https://developer.walmart.com/us-marketplace/docs/faq-getting-started)

From the pilot spec feedback doc:

- [Apple Developer Program enrollment](https://developer.apple.com/help/account/membership/program-enrollment/)
- [TestFlight](https://developer.apple.com/testflight/)
- [Google Play developer registration](https://support.google.com/googleplay/android-developer/answer/6112435?hl=en)
- [Google Play testing requirements](https://support.google.com/googleplay/android-developer/answer/14151465?hl=en)
