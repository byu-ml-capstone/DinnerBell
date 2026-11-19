# Architecture Audits

Review prompts for checking code against the principles in `architecture.md`. These are **review tools**: they find and report problems but do not fix them. Read `architecture.md` first; these prompts assume its definitions (layers, Non-negotiable rules, Decision Log).

## When to Use Which Prompts

| Situation | Use |
|-----------|-----|
| After any single change | The **Pre-Merge Checklist** in `architecture.md` (not this file) |
| Reviewing a pull request or a set of related changes | **Diff-scoped prompts** below. Run the ones relevant to what changed, or all nine for large changes |
| Milestones, before sprint reviews, or after major refactors | **Full-application prompts** below |

Running full-application audits after every change is slow and produces repeated findings. Save them for milestones.

## Rules for Every Audit

- **Do not modify code.** Report findings only.
- **Cite evidence** for every finding as `file:line`.
- **Check the Decision Log** in `architecture.md` before reporting. A deviation recorded there is an accepted decision, not a finding, unless the code goes beyond what the decision allows. Entries marked **Proposed** count as accepted until the team rejects them.
- **Rank findings by severity.** Any break of a Non-negotiable rule comes first.
- **Separate certainty from suspicion.** Mark findings you could not fully confirm (for example, behavior that depends on runtime configuration).

## After an Audit

Audit output is evidence for decisions, not an automatic rewrite request.

1. **Verify the findings.** Open the cited code. Reproduce each important claim and reject unsupported ones.
2. **Decide what to address.** Choose the most consequential risks. Fix them now, or record an explicit tradeoff in the Decision Log.
3. **Carry the lessons forward.** If an audit reveals a recurring problem, add or refine a rule in `architecture.md`.

---

## Diff-Scoped Prompts

Use these on a pull request, branch, or set of changed files. Each prompt limits the audit to the change and the code it directly touches.

**1. Boundaries**
> Review only the changes in this diff. Identify any business rule added or modified outside the service layer, split between a handler and a service, or duplicating a rule that already exists elsewhere. Check that changed handlers only parse input, call services, and map outcomes to responses. Cite file:line evidence. Do not modify code.

**2. Persistence / DAO rule**
> Review only the changes in this diff. Identify any new database connection, session use, query, or ORM operation outside DAO/repository modules. For each state change the diff introduces or modifies, check whether related writes are wrapped in one transaction and whether database constraints protect the data. Cite file:line evidence. Do not modify code.

**3. Cohesion and coupling**
> Review only the changes in this diff and assess separately. Cohesion: does the change add unrelated responsibilities to an existing module? Coupling: does it add a dependency on a concrete implementation (ORM session, model, SQL client, external API client) where a narrow contract exists or should exist, or introduce a dependency cycle? Cite file:line evidence. Do not modify code.

**4. External input**
> Review only the changes in this diff. List every new or changed input path (request fields, query parameters, files, AI/LLM output, webhooks, third-party responses). For each, check that the handler validates shape and limits, the service enforces meaning and permissions, and storage protects integrity. Identify missing or bypassable checks. Cite file:line evidence. Do not modify code.

**5. Data and access**
> Review only the changes in this diff. Check that every new or changed read and write of protected data is authorized on the server, that queries are parameterized, that no secrets are added to source or config files, and that new logs and error responses do not expose credentials or personal data. Cite file:line evidence. Do not modify code.

**6. Testability**
> Review only the changes in this diff. For each behavior added or changed, list which of these cases are tested: normal, boundary, invalid, unauthorized (where applicable), duplicate, missing, and failure. Identify untested cases, missing regression tests for bug fixes, tests that would pass even without the change, and new brittle global patches. Cite files. Do not modify code.

**7. Errors and concurrency**
> Review only the changes in this diff. For each state-changing operation it adds or modifies, identify the invariants it must preserve. Check for swallowed exceptions, raw exception text in responses, partial writes, and unsafe behavior under duplicate retries or simultaneous requests. Report which of these cases have tests. Cite file:line evidence. Do not modify code.

**8. Readability**
> Review only the changes in this diff. Identify state changes hidden inside generically named functions, workflow logic placed far from related code, functions or components that grew to span unrelated steps, and names or comments that obscure what the code does. Propose focused, behavior-preserving improvements. Cite file:line evidence. Do not modify code.

**9. Dietary safety (Dinner Bell)**
Run this whenever a diff touches restrictions, dietary labels, plan generation or editing, shopping lists, recipe editing, or any user-facing wording about restrictions.
> Review only the changes in this diff against the Dietary restrictions rules in `architecture.md`. Check that restriction checks stay deterministic code reading the strict dietary label field, with no AI in the decision. Check that every changed path that shows a conflicting recipe (proposed plan, planned meal, shopping list item) shows an alert naming the conflict, and that conflicting recipes are ranked lower, not excluded or hidden. Check that the check still reruns when a recipe is edited, an ingredient is swapped, or household restrictions change, including for approved plans. Check that label writes record their source and time, that no user-facing text says "allergen-free," "safe for," or anything similar, and that label reminders remain on recipes and shopping lists. Check that tests include the hard cases in spec §5.4. Cite file:line evidence. Do not modify code.

---

## Full-Application Prompts

Use these at milestones to audit the whole codebase.

**1. Boundaries**
> Audit the whole application for business rules. Identify rules implemented outside the service layer, split across layers, or duplicated. Cite each finding with file and line. Do not change the code.

**2. Persistence / DAO rule**
> Search application source files for database connection creation and query or ORM operations. Count a file only when it directly connects or queries. Exclude calls made through a DAO. Report each direct-access location with file and line evidence, then identify any state change that needs an explicit transaction or constraint. Do not modify the code.

**3. Cohesion and coupling**
> Audit the whole application and assess separately. Cohesion: which modules combine work that changes for different reasons? Coupling: which concrete dependencies could make a change ripple? Cite representative file/line evidence. Do not modify code.

**4. External input**
> Audit external input validation across this app. Inventory its input paths, then trace each through the UI, API handler, service/domain, and DAO/database. Check that the API validates request shape, the service enforces business rules, and storage protects data integrity. Include AI or webhook inputs where present. Identify missing or bypassable checks and cite file:line evidence. Do not change code.

**5. Data and access**
> Identify sensitive data and trust boundaries in this application. Check where the server enforces required access, where secrets come from, whether queries use parameters, and whether logs can expose credentials or personal data. Cite file and line evidence. Do not modify the code.

**6. Testability**
> Audit every important workflow and map its normal, boundary, invalid, unauthorized (where applicable), duplicate, missing, and failure cases to tests. Identify untested rules and database, network, serialization, and UI boundaries. Note repeated cases suited to parameterization, missing regression tests, brittle global patches, and CI omissions. Cite files; do not edit code.

**7. Errors and concurrency**
> Audit state-changing operations for swallowed exceptions, unsafe responses, partial writes, and concurrent or repeated requests. Identify the invariants each operation must preserve. Report existing and missing tests for injected failures, duplicate retries, and simultaneous requests. Cite file and line evidence. Do not modify the code.

**8. Readability**
> Audit the application for workflows scattered through unrelated code or buried in oversized functions or components. Find hidden state changes. Ensure names clearly describe rather than obscure behavior. Prioritize issues that hinder review or debugging; cite file:line evidence and propose focused, behavior-preserving changes. Do not edit code.

**9. Dietary safety (Dinner Bell)**
> Audit every path that reads, writes, or displays household restrictions and recipe dietary labels. Confirm there is one deterministic restriction check reading the strict label field, with no AI involved and no second implementation elsewhere. Trace every way a conflicting recipe can reach the user (plan generation, plan edits, approved plans, shopping lists) and confirm an alert naming the conflict is always shown. Confirm the check reruns on every trigger in spec §5.2. Search user-facing text for "allergen-free," "safe for," and similar claims, and confirm label reminders appear on every recipe and shopping list. Map each hard case in spec §5.4 to a test. Cite file:line evidence. Do not modify code.
