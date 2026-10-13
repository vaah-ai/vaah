# Vaah

*The agent-native platform for building next-gen agentic software.*

## Overview
Vaah is an agent-natiave HMVC software application development platform for rapid agentic software development. It is an open-source **core** that you extend with completely separate, installable **modules** to add functionality on top. The software you build on Vaah is **agent-native** — AI agents can understand and operate it through open standards (MCP, A2A, OpenAPI).

Most importantly, Vaah is designed to be the favorite of both its audiences: the platform **developers** choose to build on, and the platform **LLMs** (Claude, ChatGPT, Gemini) can drive.

## Design Priorities
Four guiding constraints for all design decisions:
- **Developer experience** — the core exists so third-party developers can build modules (and, through them, software for their businesses) quickly and safely.
- **Agent-first** — every capability is usable by agents as well as humans, via open standards (MCP, A2A).
- **High performance** — fast by default; concrete requirements in the Performance section.
- **Extremely high extendability** — modules plug into stable, documented extension points without touching core internals.

## Features
- Extremely efficient and great developer experience
- HMVC Architecture
- AI Integration as Core
- Agentic Capabilities
- Real-time dashboards
- Highly Performant
- It should support dark and light mode
- Vaah's top menu, left menu, profile dropdown on top menu and various other ui component should be extendable from other modules
- Frontend should support themes which can be and should be configurable from the admin panel Settings page
- Email and Notification should be customization using editor and it should support variables
- Notification can be delivered with various methods, like email, admin dashboard, push notification etc

## UI & Design
- Use of semantic colors (consistent tokens for primary, success, warning, destructive/error, info, muted) so theming and dark/light mode stay consistent.
- Very efficient, highly dense dashboard layout.
- Use a highly efficient font that is supported across all devices.
- Dashboard should be mobile-friendly (responsive across screen sizes).

## Developer Experience
The core is open source so that third-party developers can build modules on it — and, through those modules, software for their businesses. DX is measured by how fast and safely such a developer goes from idea to a working, distributable module.
- **Audience**: the module developer (or team) building a module to ship software for a business; the core's job is to make that fast, safe, and repeatable.
- **One-step scaffolding**: a CLI command (or the natural-language / spec-driven flow) generates a complete, runnable module skeleton — MVC, migrations, seeds, UI, permissions, and MCP tools — in one step.
- **Conventions over configuration**: a consistent, opinionated module structure that the core auto-discovers, so a developer knows where things go.
- **Local dev loop**: one-command start with a seeded dev environment and hot reload; no production infra needed to develop.
- **Spec-driven build**: constitution → specify → plan → tasks → implement → converge (see Agent-Native Platform) — a module is built audibly and repeatably, by a human or a coding agent.
- **Coding-agent friendly**: a CLI + agent-readable conventions (`AGENTS.md`) so coding agents scaffold and extend modules the way they work on any modern repo.
- **Tooling**: strict TypeScript, auto-generated types from the data model, linting/formatting, and a consistent error surface.
- **Test & verify**: a module test harness (unit + integration against a seeded instance) to verify a module before distributing it.
- **Portability**: a module built once runs on any Vaah deployment (self-hosted by different businesses) without code changes, by targeting stable core extension points (see Extension Points).

## RBAC
- Backend will have permission, roles and users
- Roles can have many permissions, permissions can have many roles
- Users can have many roles, roles can have many permissions
- Permissions cannot be deleted — they also exist at the code level.
- **Field-level permissions**: roles can grant read/write access at the column/field level, not just the entity level.
- User permissions are verified at the code level before any operation is performed.
- If a user is marked inactive or deleted, they immediately lose the ability to perform any action.
- **Agents as actors**: agents are first-class actors with their own identity, scoped credentials, and audit trail; they are subject to the same permission checks as users (see Agent-Native Platform).
- Permissions dictate what users can do and see 

## Users
- A user can be active or inactive.
- A user can have multiple statuses.
- A `notes` column for quick remarks.
- Displays the count of roles a user has, with the option to add or remove roles.
- Displays the count of permissions a user has, with a search option.
- There must always be at least one superadmin user, and a superadmin user cannot be deleted.
- A user cannot delete their own account.

### User deletion & cascade
Deleting a user is a **platform-wide event, not a single-table operation**. A user is referenced by orders, tickets, comments, audit entries, agent sessions, and every module's own tables — deleting the row alone would leave orphaned foreign keys pointing at nothing.

- **Deletion emits a namespaced event on the platform event bus** (`user.deleted`), delivered durably via the transactional outbox so no module can miss it because of a restart (see Event bus).
- **Modules listen declaratively** and apply their own cascade: soft-delete or hard-delete the rows that reference that user. *The decision is deliberately the module's, because only the module knows whether its rows are recoverable, contractually required, or referenced by something else.*
- **Two deletion modes, chosen at the point of deletion**:
  - **Soft delete** (the default) — the user is marked deleted and every module follows its own soft-delete policy; nothing is destroyed and restore remains possible.
  - **Hard delete** — the user record and all dependent module data are **actually removed and not recoverable**. This is the same irreversible operation as a privacy erasure (see Data Privacy & Consent).
- **Hard delete is confirmed and previewed**: Vaah shows what will be affected — core relations plus a per-module count of rows each module reports it will delete — before anything is destroyed. A cascade whose extent is invisible is a cascade nobody can safely trigger.
- **A module that does not implement a cascade is not blocked**, but its orphaned references are reported at deletion time rather than silently left behind.
- The superadmin guarantees above are enforced before either mode runs.

## Modules
- New modules can be installed, enabled, disabled, upgraded, downgraded, deleted, with options to run migrations, seeds, etc.
- Each module can have its own MVC structure with migrations, seeds, notifications, emails, etc. **The hierarchy is one level** — a module may contain sub-modules, but there is no recursion; nesting is organisational, not inheritance.
- Seeds are divided into two: 1. Required — when a module is activated, these seeds must run; 2. Sample — these seeds contain sample demo data.
- **Seeds are idempotent and duplicate-checked**: each seed declares how it identifies its records (a natural key or conflict column) so re-running updates rather than duplicates. Re-activation re-runs seeds safely, and **reset** removes sample data along with migrations and associated media/taxonomies.
- When a module is activated, it runs the migration and required seeds.
- **Extending workflows and agents**: a module's seeder can register workflows and agents into the platform (each stored as a record tagged with its owning module); they follow the module lifecycle — created/enabled on activation, updated on upgrade, and disabled/removed on deactivation or deletion.
- **Detach on edit**: once a user edits a module-seeded workflow or agent, it detaches and becomes user-owned — it no longer follows the module's lifecycle and survives the module's downgrade or deletion. A clear "owned by <module>" indicator shows the origin.
- **Distribution (later stage)**: a module registry / marketplace for distributing modules — publishing, discovering, and installing (distinct from the frontend module registry used for UI extension) — hosted at vaah.ai, planned for a later stage; until then, modules are installed directly (e.g. from a package/URL).
- **Modules are self-contained packages** — installed as archives (zip) or via npm, each carrying its own build output and assets, frontend bundle, migrations, permissions, and manifest. Installing a module is a data operation, not a rebuild of Vaah. **Before activation Vaah runs sanity checks**: valid manifest, satisfied `vaah_compatibility`, dependencies installed and active with version ranges satisfied and no circular dependencies, no conflicting host-supplied dependency versions, migrations with up-paths, no permission or route collision, loadable frontend bundle. Activation is fail-closed and dry-run first. Vaah checks that a module is *correctly formed*, not that it *works* — it does not run module tests (see Module trust & the marketplace).
- **Declared capabilities are shown before install** — network access, storage access, background jobs, routes and menu entries, agents and their scoped tools, and the supported Vaah version range — with the operator's approval recorded in the audit log. Contributing an agent is surfaced separately and cannot ride along silently.
- **Trust model**: as with WordPress plugins, Vaah verifies registration and then gets out of the way — no signing, no author verification, no sandboxing, no quality gating. A module can pass every check and still be malicious or broken; the operator assumes that by installing.

## Licensing & Distribution
- **Core is open source**: the Vaah core platform ships open source; commercial value lives in modules.
- **Independent module licensing**: each module carries its own license — open source, freemium, premium, or SaaS.
- **Mixed-licensing runtime**: the core installs, enables, upgrades, and runs modules of different license types side by side; license type is a per-module property.
- **Entitlement enforcement**: the module lifecycle supports license checks — premium modules verify a license/entitlement, freemium modules enforce feature or seat caps, SaaS modules authenticate to a remote service. The core provides the enforcement hooks; billing/entitlement logic stays with the module/vendor.
- **No core metering/billing**: the core does not meter or bill (BYO LLM key, see Settings).
- **Isolation**: a module's code and proprietary/licensed logic stay contained within the module (HMVC); the core exposes stable extension points, not module internals.

## Extension Points / Module Extension Model
The core exposes a stable, documented set of extension points so a module (built by a third-party developer) plugs in cleanly. This is "extremely high extendability" made concrete:
- **UI shell**: contribute top-menu items, left-menu items, the profile dropdown, routes/pages, and dashboard widgets (via the frontend module registry).
- **Core screens**: extend an existing Vaah screen by filling a **named, core-declared UI extension slot** — a record-detail tab, a list-view column, a record-page action button, or a related-items panel. Modules cannot create or inject into undeclared locations. See the UI Extension Slots section.
- **Data**: add entities via migrations; required + sample seeds; use the shared storage layer for files.
- **RBAC**: register permissions and roles for the module's entities/actions (see RBAC).
- **API + MCP**: contribute REST endpoints and MCP tools, reusing the shared service layer so RBAC, audit, and attribution apply automatically.
- **Workflows**: contribute triggers, actions, and step types to the automation engine (see Workflows).
- **Agents**: contribute named agents with scoped tools and permissions (see Agents).
- **Settings**: contribute entries to the admin Settings page.
- **Notifications / email**: register notification types and email templates via the platform's notification system.
- **Events**: emit/subscribe on the platform event bus, which drives workflow triggers and real-time updates. Redis-backed; events are namespaced and typed; module subscriptions are declarative (a module declares what it listens to, making its event surface visible and auditable rather than hidden in code). Durable consumers (workflow triggers, email) receive events via the transactional outbox; in-process UI subscribers are at-most-once and self-correcting. See the Event bus section.
- **Frontend UI**: modules ship runtime-loadable bundles and register routes, menu items, and components at runtime — installing a module does not require rebuilding Vaah. The host supplies shared dependencies (React, design tokens, the query client); a module must not bundle its own copy. A module whose UI fails to load is skipped and disabled like any other module fault (see Modules). Compatibility is checked before load. See the Frontend module loading section.
- **Core capabilities**: consume — and where designed, extend — media, taxonomy, audit, logging, and reports.
- **Cross-module data access**: a module reaches data it does not own through core's exposed services, not by querying another module's tables. See Cross-module service access.
- **Lifecycle**: install/enable/disable/upgrade/downgrade/delete, following the module lifecycle (see Modules).
- **Stability**: these extension points are stable across versions (semver); modules never reach into core internals (see Licensing → Isolation).

### Cross-module service access
A module frequently needs data it does not own — a Blog module showing its posts beside a user, a CRM module referencing a User. How that access is granted is the difference between an extensible platform and a codebase nobody can change safely.

- **Core exposes reusable services as the shared access surface**: services, providers, adapters, and interfaces published for module consumption — users, roles/permissions, media, taxonomy, audit, notifications, workflows, and the reporting query builder. A module calls these; it does not reach into core internals or another module's tables.
- **Access is through the shared service layer**, so RBAC (including field-level), audit, and agent attribution apply automatically to cross-module reads — exactly as they do for the UI, the REST API, an agent, or a workflow (see Agentic Platform → Security boundary). *This is the payoff of the single-service-layer design: a module gets governance for free instead of reimplementing it.*
- **Module-to-module access is declared, not implicit.** Where a module genuinely needs another module's domain data, that dependency is **declared in the module manifest** — the same `requires` mechanism used for module dependencies — and checked at activation by the sanity checks (see Modules). Declared means discoverable, auditable, and checkable before install.
- **An undeclared module-to-module dependency is not silently permitted.** A module reaching for tables it did not declare is outside the supported contract, and the operator is not protected by the compatibility promise if that module is later upgraded or removed.
- **Read access is the norm.** Write access across module boundaries requires an explicit grant in both modules' manifests — making it deliberate, reviewable, and symmetric. *A module that can silently write into another module's data can corrupt it in ways neither author anticipated.*
- **Fallback behaviour on absence**: a module whose declared dependency is uninstalled or inactive **cannot be activated** — it stays installed, and the UI names exactly what is missing or inactive. Modules are installed and removed independently; absence is a normal state, not an error — the operator installs or reactivates the dependency, and the module activates on the next boot.

## Module Architecture
The physical shape of core and modules — what a module is on disk, what it declares, and how that declaration becomes routes, schema, agent tools and admin UI. Extension Points defines *what* a module may extend; this defines *how* that extension is expressed and loaded.

- **A module is one self-contained directory.** Manifest, migrations, seeds, backend code, admin UI, and everything else live together. Portability is what makes a marketplace possible, so everything a module needs has to travel with it.
- **Modules are not a separate category from the application — they *are* the application.** Core is what remains after modules load. First-party and third-party modules are therefore **identical in shape and loaded by identical code**, differing in exactly one manifest field (`source`: bundled / marketplace / local). Splitting core from modules by location would give first-party code a privileged path the loader never exercises, and a broken loader would then only surface where someone had installed something. **Core dogfoods the loader**, so a defect is visible in every install immediately.
- **The asymmetry is capability, not location.** Core holds capabilities a module cannot have: the module runtime, contract assembly, the permission engine, and the UI extension slot registry — only a core change creates a slot. That is enforced by the extension contract, not by directory boundaries.

### The module declaration
- **One declaration per module is the single source for everything derived from it**: HTTP routes, the OpenAPI schema, MCP tool definitions, and the TypeScript shapes the admin consumes. One list, four outputs. This is what keeps an agent's discoverable surface and the UI's callable surface from drifting apart.
- **The declaration is data, not code.** The admin reads it over the API rather than importing a module's TypeScript — a React app cannot import a NestJS module class, and coupling the admin's build to every module's source would break the isolation the module model depends on.
- **Every route declares the permission it requires.** A route that cannot name its access requirement is a route that bypasses the engine, so the access field is required rather than optional. This is what makes field-level filtering enforceable *before* a query is built (see RBAC), and therefore what allows reports to be permission-governed rather than reserved to superadmins.
- **Module routes are namespaced under `/modules/<name>/`.** Two modules then cannot claim the same path, uninstalling one is a clean namespace revocation rather than an archaeology exercise, and a log reader can tell module surface from core surface. The uglier path is the honest one.
- **The declaration also carries the lifecycle fields already decided**: `core_compatibility` as a range, `requires` for dependencies, cross-module read and write grants, entities, events, contributed services, and UI slot fills (see UI Extension Slots).

### Layout
```
backend/src/
  core/            module runtime, contract assembly, permission engine,
                   slot registry, events, jobs, audit
  modules/         every module, first-party and third-party, identical shape

modules/<name>/    installed third-party modules
  module.json      manifest
  migrations/  seed/
  src/             routes, entities, services, controllers
  ui/              the module's admin components

admin/src/
  api/             the internal client — one error shape, one response envelope,
                   concurrency handling, the only supported export path
  ui/              design system and namespaced tokens
  shell/           app shell, auth gate, slot rendering
  core-pages/      Vaah's own screens
  runtime/         registry fetch, runtime-assembled router, slot renderer,
                   component map
```
- **`core/` and `modules/` are separated by trust, not by kind.** `core/` is the trusted kernel that defines the extension points; `modules/` is everything that plugs into them. First-party modules use the identical directory layout as third-party ones and have no privileged internal path.
- **`admin/src/runtime/` is separated from `core-pages/` because it cannot be statically imported.** Core pages compile in; module routes are only known once the registry is fetched. This boundary is a direct consequence of restart-scoped, data-driven activation and should be explicit rather than incidental.
- **The two registries are consumed at different tiers.** Core routes are compile-time typed, because they ship with Vaah. Module routes are fetched from the declaration at runtime and validated there — so a module adding a route never requires rebuilding the admin.
- **The admin is TypeScript, and the generated contract is a build dependency of it.** That is the mechanism that stops the admin hand-rolling API calls and reimplementing envelope discipline, concurrency and export handling per screen. In JavaScript there is no version of that guarantee.

### One boundary that must stay contained
- **How a module's UI reaches the admin is confined to two files** — the component map in `admin/src/runtime/` and the loader in `modules/<name>/ui/`. Every other part of this layout is stable whichever answer that question takes (see Frontend module loading). If it starts leaking into the router, the contract layer or the module lifecycle, the architecture has been drawn wrong and should be revisited rather than patched.

## CRUD Scaffolding
Whenever a CRUD is scaffolded, it automatically includes:
- **AI-assisted scaffolding**: a new module or CRUD can be scaffolded from a natural-language description (vibe coding) via the spec-driven build-time workflow (see Agent-Native Platform).
- **RBAC**: permissions and roles generated for that entity.
- **Migration** with standard columns: `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by`, `is_agent`, `agent_identity`, and an active/inactive status.
- **Agent attribution**: when an agent performs a change on behalf of a user, the record stores both the accountable principal (`created_by`/`updated_by`/`deleted_by` = `user_id`) and the instrument (`is_agent` + `agent_identity`, plus a link to the agent session/trace). `is_agent` reflects the most recent mutation. Applies to create, update, and delete.
- **Agent attribution in UI**: agent-made changes are identifiable on the dashboard — a row-level badge, a filter for agent-made changes, and attribution in the item/audit detail ("by Agent X on behalf of User Y").
- **API endpoints**: CRUD endpoints generated for the entity.
- **MCP**: the entity's operations are exposed as tools on the Vaah MCP server.
- **Soft delete as default** (via `deleted_at`), not hard delete. *Business deletes only — a privacy erasure hard-deletes regardless of state (see Data Privacy & Consent).*
- **Optimistic concurrency**: every mutable record carries a version; an update states the version it read and is **refused on conflict** rather than silently overwriting. On conflict the UI shows both versions and asks which to keep. Agents obey the same rule — a stale agent write conflicts and re-reads rather than clobbering.
- **UI actions**: restore a deleted record, and permanently delete it. *Restore is unavailable for records that have been erased under a privacy request.*
- **Form actions**: save, save and new, save and clone.
- **Filters**: include trashed, only trashed, only inactive, only active.
- **Sorting**: multiple sort options.
- **Views**: efficient UX for list view, item view, and a quick item view (modal).
- **Trashed items** highlighted in light red.
- **Search**: debounced search plus full-text search.
- **AI form fill**: option to auto-fill the create or edit form.
- **AI search**: AI-powered search.
- **Real-time**: an edit by one user reflects immediately to all users.
- **Bulk import** — **on by default for every scaffolded CRUD**, not an add-on:
  - File-based: CSV, and JSON; spreadsheet formats supported where practical.
  - **Column mapping**: imported headers are matched to fields automatically, and mismatches are shown for correction before anything is written. *Import fails on a silent guess — a column mapped to the wrong field is a data-integrity bug that is very hard to find afterwards.*
  - **Validation before commit**: the file is fully parsed and validated first, and errors are reported per row so the operator fixes the file rather than discovering problems halfway through.
  - **Partial vs. all-or-nothing** is the operator's choice, stated explicitly — a half-applied import is the outcome most likely to be misunderstood later.
  - **Existing records** are matched by a chosen key (id, email, or a natural key) so re-importing updates rather than duplicates; the collision policy is shown before the run.
  - Runs as a **queued job** with progress, so a large import never times out an HTTP request, and it is **audited and attributed** like any other bulk operation (see Audit Logs).
  - **Permissions apply per row, not just to the file** — an import cannot write a record the importing user could not create through the UI, and an agent-driven import is attributed as one.
- **Bulk export** — the inverse: the current filtered set exported to CSV/JSON, respecting field-level read permissions, rate-limited and audited for the same reasons as import.

## Audit Logs
- Global audit logs for every record and settings change.
- Audit logging is a core capability that modules can use or extend.
- A dedicated page to view all logs, with various filters.
- **Agent attribution & bulk operations**: audit entries record dual attribution — the user principal, the agent instrument, and a link to the agent trace. A bulk agent action produces one operation-level entry (e.g. "Agent X, on behalf of User Y, disabled 214 users in country Z") linking to the affected records, in addition to the per-record entries.
- **Entries record what changed**, not only that a change occurred: before/after values for the affected fields, with secrets redacted to presence-and-fingerprint.
- **Reads are audited at operation grain**: who ran which report or query, with what filters, when, and the resulting row count. Individual row reads are not logged.
- **Audit logging fails open**: if the audit write fails, the operation proceeds and the failure is logged as an error and alerted. Audit completeness never takes the platform down.
- **Retention**: audit entries are purged after a configurable retention period applied **independently of the data-deletion workflow**. The operator sets it. For the duration of that period, personal data remains in the audit log after an erasure request — this is the documented boundary of the GDPR-ready claim (see Data Privacy & Consent).
- **Scope note**: the audit log is an operational and diagnostic record, not an evidentiary one — it is an ordinary application table and is not tamper-proof. Deployments requiring evidentiary-grade audit must scope integrity themselves.

## Settings
A settings page (admin) to configure:
- **AI features**: enable or disable.
- **LLM details**: configure the LLM provider, model, and credentials used by the agent.
- **Date & time format**: a global format used across the platform.
- **Custom user profile fields**: add custom fields to extend the user profile.
- **2FA**: option to enable two-factor authentication.
- **Social login**: configure providers (Google, Facebook, LinkedIn).
- **Upgrade**: option to upgrade Vaah (core).

## Registrations
- A `registrations` CRUD.
- Frontend registration/signup stores data in `registrations`, not directly in `users`.
- On successful verification, the record moves from `registrations` to `users`.
- Purpose: keep unverified/dummy data out of the `users` table.

## Media & Taxonomy
- **Media manager**: a core capability for managing media (upload, organize, reference) with on-demand transformations (resize/reformat) and CDN-ready delivery — usable in Vaah and its modules. Transformations are a core service behind a storage-agnostic interface, run as **queued jobs rather than inline in the request**, and keep originals immutable with transformations derived and regenerable. Uploaded files are treated as untrusted input: type and content are validated on upload.
- **Nested taxonomy manager**: a core capability for managing nested/hierarchical taxonomies (e.g. categories with subcategories) — usable in Vaah and its modules.

## Data & Storage
- **File/media storage backends**: the platform stores files on one of two backends — **local filesystem** (files on the server) or **S3-compatible object storage** (AWS S3, MinIO, DigitalOcean Spaces, Cloudflare R2, …).
- **Storage abstraction**: the media manager and all modules read/write files through a storage layer that is backend-agnostic, so the backend is a configuration choice, not a code change.
- **Chosen at onboarding**: the storage backend (local vs S3) is selected in the onboarding/setup wizard; for S3, the endpoint, credentials, and bucket are captured at that point. The choice is stored as platform configuration.

## Notifications & Email
The platform's notification and email system — used by the core, modules, workflows, and system alerts.
- **Notification model**: a notification is a typed message targeted at a user or a set of users (e.g. a role), with a payload and the channels to deliver it on.
- **Channels**: in-app (admin dashboard), email, and push notification; SMS/webhook as optional additions. A notification can fan out to multiple channels.
- **Email sending**: transactional (single, immediate) and bulk (queued) email; delivered via **SMTP** (self-hosted) or an API provider (AWS SES, SendGrid, Postmark). SMTP/provider credentials are held by the backend (see the security boundary — agents/workflows never hold SMTP credentials).
- **Templates**: notifications and emails are driven by templates edited in a visual editor, supporting variables (e.g. user name, record values) and versioning; the email/notification customization from Features lives here.
- **Preferences**: per-user channel opt-in/out (and, where relevant, quiet hours).
- **Reliability**: delivery is async via the queue (Redis), with retries, delivery/bounce/complaint status, and rate limiting to prevent spam/abuse.
- **Extension**: modules register their own notification types and email templates (see Extension Points); system alerts (e.g. job failure, health degradation) reuse this system (see Observability).

### Email deliverability & bounce handling
**Sending email is Vaah's responsibility; being trusted by receiving inboxes is not.** That boundary is drawn deliberately, and it matches the deployment model — under single-tenant self-hosting, the operator owns the sending domain and the reputation attached to it.

- **Not Vaah's responsibility**: SPF, DKIM, and DMARC records, domain reputation, and inbox placement. These live with whoever owns the sending domain — which for a self-hosted deployment is the operator, configured at their DNS registrar. **Vaah does not manage, publish, or verify DNS records**, and a deployment that sends from an unconfigured domain will simply land in spam, which is the correct and predictable outcome.
- **Vaah's responsibility**: send correctly and know what happened. Delivery attempts, bounce, and complaint status are recorded per message and surfaced in a **delivery log** (per recipient: sent, delivered, bounced, or complained, with the provider's reason), so an operator can answer "did this mail actually go out?" without leaving Vaah.
- **Bounced addresses are managed, not ignored.** A **suppression list** holds addresses that hard-bounced or complained; sending to a suppressed address is skipped and logged rather than retried. *Silently continuing to mail a dead address degrades the sender's reputation for everyone else on the deployment, so suppression protects the operator, not just the recipient.* Suppression is reversible — an operator can clear an entry.
- **Transient vs. permanent bounces are distinguished.** A temporary failure (full mailbox, greylisting, 4xx) is retried within the job's declared policy; a permanent failure (5xx, nonexistent mailbox) is not retried, is recorded, and is surfaced as a dead-letter (see Resilience & Failure Handling).
- **A configured bounce/complaint webhook address** lets the provider report asynchronously; polled status remains the fallback so bounce handling does not depend on the webhook being reachable.
- **An unusable configuration is visible.** If the sending domain is unauthenticated or the SMTP connection fails, Vaah surfaces that as a health-check warning rather than failing silently into a queue that quietly backs up.

## Reports
- Configurable reporting/BI sections (like Metabase): build dashboards, charts, and reports over platform data via configuration, not code.
- Reports can be shared, scheduled, and embedded in dashboards or modules.
- **The reporting surface is superadmin-only.** Composing an arbitrary query over platform data is not something the field-level permission model can police at runtime — a query builder lets a user assemble filters and columns the service layer never saw in advance. Rather than weaken field-level permissions or bolt an unprovable filter onto the builder, Vaah restricts the surface to the one role that is exempt by definition.
- **This is a deliberate narrowing of an otherwise useful capability**, and the honest trade: reporting is powerful enough that giving it to a normal role would make field-level RBAC decorative. Non-superadmin users work from curated reports and the standard list/detail views, which are permission-checked per field.
- **Scheduling a report does not change who may read it** — a scheduled report is delivered under the run-as identity it was created with and honours the recipient's own read permissions on delivery. A report is never a way to route data around a permission.
- Report runs remain audited (see Audit Logs).

## Workflows / Automation
A deterministic automation engine: configure trigger → steps → actions without code. This is the "workflows" half of the workflows-vs-agents principle (see Agent-Native Platform) — fixed-path operations run as workflows; open-ended ones run as agents.
- **Triggers**:
  - Event: on record created/updated/deleted (per entity), on a custom event, a settings change, or a user action.
  - Schedule: cron-like (daily/hourly/weekly), interval, or one-time.
  - Manual: from the UI (a button) or via API.
  - API / MCP: fired through the API or as an MCP tool, so agents can start workflows.
- **Steps / actions**: CRUD on records; call an internal or external API; send a notification or email (uses the platform's email system); run a sandboxed script; emit a webhook (outbound); and **invoke an agent** for a step (a deterministic workflow with an AI step).
- **Control flow**: conditions/branching, loops over record sets, parallel branches, wait/delay (time or until an event), and sub-workflows.
- **Authoring**: a visual step builder (configuration, not code) and/or config (YAML/JSON) for developers. Agents can also author workflows (vibe-coded) and trigger them.
- **Execution model**: a deterministic executor (no LLM) runs the steps in order; each step calls the shared service layer — the same one that enforces RBAC, audit, and agent attribution for every caller. Workflows never hold infrastructure credentials (DB, SMTP, …).
- **Execution identity**: each workflow runs under a configurable run-as identity — event-driven workflows default to the triggering user, scheduled workflows use a system/automation identity; the identity is recorded in audit and subject to RBAC.
- **Observability**: every run is logged (trigger, per-step results, duration, outcome, records affected); a dedicated runs page with filters (like the audit page); real-time status; pause/stop a running workflow.
- **Error handling**: retries, on-failure notification, and idempotency.
- **Governance**: RBAC over who can create/edit/run a workflow; every run audited; agent-triggered/created workflows record `is_agent`/`agent_identity`; workflows are versioned and can be enabled/disabled and rolled back.
- **Modules**: modules can contribute their own workflows (seeded via their seeder, following the module lifecycle) as well as workflow triggers, actions, and step types (extensible, like the module MVC).

### Visual builder
- **Approachable editor**: a top-to-bottom sequence of steps (Zapier-style) with branch (if/else) and loop blocks — not a free-form node canvas. Easier for admins to read and reason about.
- **Action palette**: a searchable palette of available steps (CRUD, API call, email, agent-invoke, webhook, script, sub-workflow); drag onto the canvas.
- **RBAC-aware palette**: it only shows actions the current user is actually allowed to perform.
- **Visual data mapping**: map one step's output to the next step's input via field-pickers/dropdowns — the "data flow" UI.
- **Condition & loop editors**: visual if/else (field · operator · value) and "for each record in [set]."
- **Live validation**: inline errors for missing fields, bad mappings, or permission problems as you build.
- **Dry-run / test**: run against sample data or a specific record to preview the effect without committing.
- **Templates**: pre-built starter workflows (new-user onboarding, daily digest, …) to start from.
- **Agent-assisted build**: describe the workflow in natural language → the agent proposes the graph → you review and adjust in the visual editor.

## Onboarding
- An onboarding/setup wizard (similar to WordPress) run on first launch to configure the platform:
  - Database connection
  - Storage backend (local filesystem or S3-compatible)
  - Run migrations
  - Run seeds
  - Create the admin account
  - Other initial setup steps

## Agentic Platform
- An AI button in the admin dashboard opens a chat where the user asks the AI agent to perform a task; the agent performs the task and renders UI components in the chat.
- **Agent runs server-side (in NestJS)**: the agent loop (LLM + tool orchestration) executes in the backend, not the browser — keeping LLM keys and tool execution off the client, enforcing the security boundary, and enabling scheduled/autonomous and background agents to run without a client. The dashboard chat is a thin client that streams the agent's output (tokens, tool progress) over a persistent connection.
- The agent is backed by two MCP servers:
  - Vaah MCP server: CRUD and domain operations (generic tools + schema resources + journey prompts), reusing the same backend services as the REST API.
  - GenicUI MCP server: UI rendering (render / update / subscribe), provided by GenicUI and used as-is (not re-implemented in the Vaah server).
- **Shared identity across both servers**: the domain and UI servers act under one shared identity — the same user token, RBAC, audit, and `is_agent`/`agent_identity` attribution. The UI server never runs with a different or elevated identity than the domain server; the agent is the intermediary (the servers do not call each other).
- Component model (chosen per capability):
  - Capability components: self-contained, own business logic, semantic props (e.g. UserSearch accepts a query prop and fetches its own data).
  - Presentational components: agent-fed data (e.g. UserDetail receives user data from the agent).
  - Components are authored in React (reused from the dashboard) and wrapped in a thin Web Component (a `GenicElement` subclass that renders the React component into a closed Shadow DOM) for GenicUI; the same React components are packaged as MCP Apps for cross-host rendering. One React component library, two packaging targets.
- Component catalog: every registered component carries a when-to-use description, a prop schema, and an example, so the agent can discover and route to the right component.
- **UI rendering surfaces (dual)**:
  - **In-app**: GenicUI renders components inside the Vaah dashboard chat (deep AG-UI integration, shared state, theming, real-time).
  - **Cross-host**: the same components are packaged as MCP Apps (an extension to the core MCP spec) so they render inside any MCP host (Claude Desktop, VS Code Copilot, M365 Copilot, …).
  - Phased: GenicUI first (product surface), MCP Apps as a later portability layer.
- Agent security:
  - RBAC propagates into every tool call — the agent acts as the logged-in user and is subject to the same permission checks as the REST API.
  - Idempotency keys on create/update operations.
  - Audit log of all agent-driven mutations.
  - Destructive operations are gated behind a confirmation UI.
  - Data returned by operations is treated as data, never as instructions (prompt-injection defense).
  - **Security boundary**: agents and workflows never hold infrastructure credentials (DB, SMTP, …). All actions execute through the shared service layer, which enforces RBAC, audit, and agent attribution for every caller — UI, API, agent, or workflow.

## Agents
The platform supports multiple named agents, each a pre-configured specialist with a defined role, scope, and behavior. The general chat agent is the default agent; the others are specialized. An agent is a concrete instance of an "agent actor" (see RBAC, Agent-Native Platform).
- **Definition**: name + role description + system prompt/instructions + a scoped tool allow-list (which MCP tools it may use) + a scoped RBAC/data boundary + model/provider config (or inherit from Settings).
- **Scope & permissions**: each agent has its own identity and a scoped permission set — it can only do what its role allows (e.g. a Support agent can read users and create tickets, but not delete records).
- **Invocation**: by name ("ask the Reporting agent" / a slash command); an agent can be the default for a context (e.g. the Reports section uses the Reporting agent); and it can be invoked by a workflow step.
- **Autonomy levels**:
  - Interactive — acts only when the user prompts, within its scope.
  - Scheduled / autonomous — runs on a schedule or trigger within its scope (bridges to Workflows).
  - Sub-agent — can be delegated to by another agent/orchestrator (ties to A2A + sub-agents).
- **Governance**: RBAC over who can create/configure/invoke; every action attributed (`is_agent`/`agent_identity` records which agent) and audited; per-agent cost caps; enable/disable.
- **Built-in agents**: the platform ships defaults (general assistant, reporting, data entry); modules can contribute their own via their seeder (e.g. CRM adds a Sales agent), following the module lifecycle.

## Agent Operations
- **Agent observability**: a view of what the agent did — full trace of tool calls and reasoning, token/cost per action, conversation/session history, a stop/kill control, and feedback.
- **Cost control**: budgets and rate limits — per-user, per-module, and per-session token and spend caps, with the ability to stop the agent when a budget is exceeded.
- **Streaming**: token-by-token responses and live "agent is calling tool X" progress in the chat.
- **Model provider abstraction**: multiple LLM providers with routing and failover (see Model Management & Guardrails).
- **Data privacy / PII policy**: explicit rules on what user data is sent to the LLM, PII/secret redaction, and data-residency options (see Model Management & Guardrails).
- **Secrets management**: LLM keys and other secrets are stored encrypted and rotatable (see Security & Data Protection).
- **MCP management UI**: view/enable/configure MCP servers and tools, connect external MCP servers, and allow-list which tools a role can use.

## Agent-Native Platform
Vaah is agent-native: it is designed so that AI agents — the built-in runtime agent, external coding agents, and other systems' agents — can understand and operate it with minimal hand-holding, each via an open standard.

### Three agent surfaces
- **Runtime** — operate the live app: the built-in chat agent performs CRUD/domain operations and renders UI (see Agentic Platform, Agent Operations).
- **Build-time** — develop on the platform: coding agents (e.g. Claude Code, Copilot) scaffold and extend modules, CRUDs, and custom code on Vaah.
- **Inter-agent** — agents delegating to/from Vaah: external agents treat a Vaah agent as a peer.

### Full MCP surface (Vaah MCP server)
Beyond tools, the server exposes the full MCP capability set:
- **Tools**: CRUD/domain operations, reusing the same backend services as the REST API.
- **Resources**: the data model, schemas, and documentation exposed for progressive disclosure. Per-entity schemas use resource templates (parameterized URIs); annotations (`audience`, `priority`) steer the model.
- **Prompts**: reusable multi-step "journeys" exposed as slash commands.
- **`instructions`**: server-level standing guidance returned at initialize, describing how to use the whole server (tool specs as prompts).
- **MCP Apps**: Vaah components can render as sandboxed-iframe HTML apps inside any MCP host (Claude Desktop, VS Code Copilot, M365 Copilot, …) via a `ui://` resource, with bidirectional tool/data flow. MCP Apps are an **official MCP extension** (not core), and **the Vaah MCP server must be fully functional when a host ignores the `ui://` resource** — every piece of data the model needs is also present in the plain tool result. *Client support is uneven and the extension is a known attack surface, so the tool result is the contract and the rendered app is the enhancement.* See D-2.
  - **Stateless protocol**: the current MCP core is stateless — no `initialize` handshake and no session ID. Authentication and context travel as headers on every request, and the Vaah MCP server holds **no protocol session state**; all state lives in Vaah's database and in the identity carried per request.

### A2A (agent-to-agent)
- Vaah exposes itself as an A2A agent with an **Agent Card** (a JSON manifest of capabilities, skills, and connection info served at a well-known URL) so external agents can discover it and delegate tasks.
- Supports the A2A task lifecycle, including an **`input-required`** state for native human-in-the-loop.
- Transports: JSON-RPC 2.0 over HTTP(S), with SSE streaming and async push notifications.
- Complements MCP: MCP lets an LLM use Vaah's tools; A2A lets another agent delegate a whole task to a Vaah agent as a peer.

### Spec-driven build-time agents
- "Vibe coding" a module/CRUD follows a spec-driven workflow: **constitution → specify → plan → tasks → implement → converge**, each artifact reviewed before the next, with a verify loop until converged.
- A **CLI** and agent-readable conventions (an `AGENTS.md` / `VAAGENTS.md`) let coding agents scaffold and extend Vaah projects the way they work on any modern repo.
- Build-time agent work is auditable and repeatable — spec/plan/tasks/convergence reports are persisted — not a freeform prompt.

### Self-description layer
- **`llms.txt`** for the docs/website: a compact, agent-oriented overview plus links to Markdown page twins.
- **OpenAPI 3.0.x + JSON Schema** for the REST API: machine-readable endpoint and data discovery (see API Design Conventions).
- **MCP resources** for the data model.
- Goal: an agent should be able to answer "what can I do here?" from the platform itself.

### Agent as a first-class actor
- Agents are modeled as a distinct actor type (not only "the logged-in user"): their own identity, scoped credentials, and audit trail, so an agent can hold delegated authority independently of a human session (needed for A2A, background agents, and sub-agents). See RBAC.
- **Dual attribution**: when an agent acts on behalf of a user, the user is the accountable principal (`created_by`/`updated_by`/`deleted_by`) and the agent is the instrument (`is_agent` + `agent_identity`) — recorded on every affected record (see CRUD Scaffolding).

### Workflows vs agents (routing principle)
- Deterministic, fixed-path operations (e.g. scaffold a CRUD) run as **workflows**; open-ended tasks (e.g. vibe-coding a new module, debugging) run as **agents**.
- Route by task type; add autonomy only where it measurably improves outcomes.

## Model Management & Guardrails
The LLM layer that agents and AI features use is centrally managed, provider-agnostic, and guarded.
- **Providers & BYO keys**: supports multiple LLM providers; keys are BYO, stored encrypted (see Security & Data Protection), and configured at the instance level (see Deployment model).
- **Model routing & fallback**: select a model per task/feature (e.g. a fast model for extraction, a capable model for reasoning), with automatic fallback to a secondary provider/model on error, timeout, or rate-limit.
- **Prompt management**: system prompts and tool instructions are versioned, editable, and testable; changes are auditable and reversible.
- **Guardrails**:
  - **Input**: prompt-injection defense (data returned by operations is treated as data, never as instructions — see Agentic Platform); PII redaction before content is sent to a model where configured.
  - **Output**: model output is validated against the expected schema/contract before it drives an action; malformed or unsafe output is blocked or sanitized.
  - **Actions**: agents can only act through the shared service layer (RBAC, audit, attribution) — a model response can never grant itself privileges (see Agentic Platform, RBAC).
- **Telemetry**: per-operation token/latency/cost telemetry feeds Observability; the core does no metering or billing (see Licensing & Distribution).
- **Pinning & recording**: the resolved model (provider and version) is recorded with every agent run and trace, so a behaviour change is attributable to a specific model rather than to "sometime recently". Pinned in configuration, not merely inferred at runtime.
- **Fallback is a per-deployment configuration**: automatic fallback to a secondary provider is protective during an outage but can silently change behaviour on a provider regression — so an operator chooses. Because keys are BYO, the operator owns the provider relationship; Vaah owns the routing and the recording.
- **Model-agnostic**: no lock-in to a single provider; the model layer is swappable without changing the rest of the platform.

## Agent Evaluation
An agent's output is non-deterministic, so the same prompt yields a different answer run to run. Without a quality bar, "did this change break the agent?" can only be answered in production — and because agents hold real permissions, a behavioural regression is a *safety* event, not just a quality one.
- **Versioned offline eval suite**, runnable without a deploy, in two parts:
  - **Golden tasks** — fixed scenarios with expected outcomes ("given these records and this instruction, the agent filters by status and touches nothing else"). The regression net.
  - **Red-team set** — prompt injection, data exfiltration, permission escalation. The guardrails promise these defences; the eval proves they still hold after a change.
- **Assertions are invariants, not exact outputs.** Evals assert that the right records changed, that the wrong ones did not, that no permission was escalated, and that guardrails held. *Weaker than a unit test, and the strongest thing available for non-deterministic output.*
- **Gating**: changes to prompts, tool definitions, or model configuration are blocked on eval results; other changes are advisory. *A noisy gate trains people to bypass it and is worse than none — a flaky eval must be fixed or demoted, not tolerated.*
- **Prerequisite for diagnosis**: model versions are pinned and recorded per run (see Model Management & Guardrails), so a regression is attributable to a specific model.
- **Core owns the framework; the CLI scaffolds eval files for modules** — Vaah defines the contract and generates the starting point, and does not gate module behaviour (consistent with its position on module conformance generally).

## Platform
- **Deployment model**: single-tenant — one deployment serves one business; see the dedicated Deployment model section.
- **Localization (i18n)**: see the dedicated Localization section.
- **Accessibility**: see the dedicated Accessibility section. **Target is WCAG 2.2 AA, enforced as a blocking CI gate** (see Accessibility).
- **Visual regression testing**: **advisory, not a blocking gate** — reported on every change with a link to the diff so a reviewer can see what moved. *Contrast with accessibility: an accessibility failure excludes users and is objectively checkable; a visual diff is often a subjective two-pixel reflow, and a blocking gate on those trains people to re-run until it passes.*
- **Data retention & GDPR**: see the dedicated Data Privacy & Consent section. Retention policies and right-to-be-forgotten / data deletion; scheduled backups, restore, and HA live in the Disaster Recovery section.
- **Webhooks / outbound integrations**: emit events to external systems.

## Deployment model
Vaah is **single-tenant**: one deployment serves one business/organization. This is a deliberate decision, not a limitation to be lifted later.
- **Rationale**: a self-hosted platform should not carry the complexity of isolation it does not need. Multi-tenancy would make every query, every backup, every GDPR deletion, and every configuration surface a cross-cutting concern.
- **Scaling to many customers**: a developer serving many businesses runs **many Vaah deployments**. Isolation between them is the deployment boundary, which is the strongest kind.
- **Configuration**: branding/theme, domain, settings, and the LLM provider + key (BYO) are **instance-level** settings.
- **Data**: one database. Backup, restore, and deletion are operations on that one database.
- **Roles/permissions/users**: not tenant-scoped — they are simply the platform's roles, permissions, and users (see RBAC, Users).
- **Deferred, not abandoned**: if multi-tenancy is ever required, the intended path is **tenant-as-a-row** (a tenant identifier plus row-level scoping) rather than a database per tenant. That is a substantially smaller change and a weaker isolation guarantee; it is recorded here so the option is known, not assumed.

## Localization (i18n)
The core provides a **string-translation mechanism for UI strings**; modules reuse the same mechanism for their own UI strings. **Content translation is each module's responsibility** — the core does not translate user or module content.
- **UI string translation**: platform UI strings are externalized and translatable via a dedicated mechanism (translation files / i18n service, with a default language and fallback). **Modules always go through Vaah's translation API rather than importing an i18n library directly**, so module strings localise identically to core's and the underlying library stays replaceable. A missing translation falls back to the default language; a missing key falls back to the key itself rather than rendering blank.
- **Module strings**: modules use the same mechanism to translate their own UI strings, so a module's interface localizes the same way the core's does.
- **Content translation**: user-generated and module content is translated (or not) by the owning module; the core provides no content-translation service.
- **Language selection**: selectable per user, with a deployment-wide default.
- **Formatting**: locale-aware date/number/currency formatting, tied to the global Date & time format (see Settings).
- **Timezone: UTC everywhere, converted only for display.**
  - **All timestamps are stored in UTC** — every table, every audit entry, every job record. One storage convention means timestamps are directly comparable, sort correctly as plain values, and cannot be ambiguous.
  - **Conversion happens at the edge**: the client renders in the viewer's timezone, and the server converts only where a value must be expressed in a specific zone — a scheduled workflow's run time, a report's date bucket, an email's send window.
  - **The viewer's timezone is detected from the client**, not set per user by hand. A user who travels, or who is logged into a deployment used from two countries, sees times in the place they actually are. *This is the whole point of storing UTC: the platform never has to guess or store whose timezone a timestamp is in.*
  - **The one place the deployment timezone still matters is scheduling**: a workflow set to run at 09:00 needs a defined zone. It is resolved against the deployment's configured timezone, shown explicitly in the editor next to the time — an unqualified "09:00" in a scheduling UI is a bug waiting to happen, so the zone is part of the setting, not an assumption.
  - DST transitions are handled by the zone database at both ends — stored value is an instant, not a wall-clock time, so a workflow at 09:00 keeps meaning 09:00 local across a DST change rather than silently drifting to 08:00 or 10:00.
- **RTL**: right-to-left layout support for RTL languages.

## API Design Conventions
The public API follows current, industry-standard conventions. **REST is the canonical, versioned contract and the single read/write surface, described by OpenAPI 3.0.x.** One surface keeps field-level permissions tractable (per endpoint rather than per field across a type graph), preserves HTTP caching semantics (`ETag` / `Cache-Control` / 304), and gives clean per-operation observability and rate limiting. Dynamic query composition for the reporting surface is provided by the reporting query builder over the service layer (see Reports), not by a second API surface. The same conventions apply to module-generated APIs, so every entity's API behaves predictably.
- **REST (primary contract)**:
  - Resource-oriented URIs: `/{version}/{resource}` — plural, lowercase, hyphenated nouns (e.g. `/v1/users`, `/v1/countries/{id}/users`).
  - Standard HTTP methods: GET (read), POST (create), PATCH/PUT (update), DELETE; non-CRUD actions as POST sub-resources (e.g. `POST /users/{id}:disable`).
  - Correct status codes (200/201/204, 4xx, 5xx).
- **Versioning**: URI-based (`/v1/`); backward-compatible changes only, breaking changes get a new version.
- **Errors**: **RFC 9457 Problem Details** (`application/problem+json`) — `type`, `title`, `status`, `detail`, `instance`; clients key logic off `type`.
- **Pagination**: cursor-based (`page_token` / `next_page_token`), aligned with RFC 8288 Web Linking; optional total count.
- **Filtering & sorting**: consistent query parameters (e.g. `?filter[country]=X`, `?sort=-createdAt`).
- **Resource shape**: uniform JSON envelope — resource objects (`id` / `type` / `attributes` / `relationships`) and top-level `data` / `links` / `included` (JSON:API conventions adopted for a uniform shape).
- **Caching & idempotency**: `ETag` / `If-None-Match` (304) and `Cache-Control`; `Idempotency-Key` on create/update.
- **Rate limiting**: standard `RateLimit` / `Retry-After` headers.
- **No second query surface**: REST is the only read/write API. There is no GraphQL or equivalent — one surface, one authorization path, one set of caching and observability semantics.
- **OpenAPI 3.0.x**: the single source of truth — auto-generated from the NestJS service layer; drives client codegen, docs, MCP resources, and agent tool discovery (see Agent-Native Platform).
- **Auth**: OAuth 2.1 + OpenID Connect — client-credentials for applications, authorization-code + PKCE for users (see API Authentication). Local username/password is the always-available baseline credential; federated methods are opt-in and only offered when configured.

### Agent-friendliness
The API is designed so that AI agents can discover, call, and recover from it with minimal hand-holding — the same machine-readable surface that the MCP tools are built on.
- **Self-describing**: OpenAPI 3.0.x + JSON Schema (and the MCP resources) describe every endpoint, parameter, and response shape, so an agent can discover "what can I do here?" from the platform itself — no per-endpoint hand-holding.
- **Predictable & uniform**: consistent URIs, resource shape, filtering, and pagination mean an agent that learns one endpoint generalizes to the rest (fewer hallucinated paths/params).
- **Machine-actionable errors**: RFC 9457 `type` + field-level validation errors let an agent branch programmatically — re-auth on 401, pace/back-off on 429, correct and retry on 422 — instead of parsing free-text.
- **Safe retries**: idempotency keys let an agent retry a create/update without duplicates; cursor pagination is stable across retries (no offset drift).
- **Efficient**: `ETag`/304 and `Cache-Control` let an agent avoid re-fetching unchanged data (token/latency savings); rate-limit headers let it pace itself.
- **Consistent across surfaces**: REST and MCP tools both reuse the same service layer, so behavior, RBAC, audit, and attribution are identical regardless of which surface an agent uses.
- **Stable**: versioned, backward-compatible contracts mean agents and integrations don't break on minor changes.

## API Authentication
Vaah authenticates users through **several independent methods, any of which may be active at once**. There is no single mandated provider — a self-hosted deployment may run on local passwords alone, federate to an enterprise IdP, or use both.

**Methods:**
- **Local username/password** — always available as the baseline credential, and the method the onboarding admin account is created with. See Local password authentication below.
- **OAuth 2.1 + OpenID Connect** — authorization-code + PKCE for users (delegated, short-lived access tokens) and client-credentials for applications/integrations.
- **Enterprise SSO / IdP**: SAML and OIDC federation (e.g. Okta, Azure AD, Google Identity) for enterprise sign-in.
- **Social login**: Google, Facebook, LinkedIn (configured in Settings).
- **Personal access tokens**: long-lived, scoped user tokens.

**Federated methods are opt-in and conditional.** A method is offered only when it is configured — credentials in Settings, or environment variables at deploy time. Nothing external is required to run Vaah: with no keys set, local passwords are the whole story. This is what keeps "self-contained core" (see Deployment & Self-hosting) true — Vaah never requires an external identity provider to be usable.

Every method resolves to the **same internal principal**. RBAC, field-level permissions, audit, and agent attribution are identical regardless of how a user authenticated — federation changes how identity is established at the door, never what a user may do once inside. Linking a federated identity to an existing local account is a first-class operation (matched on verified email), so a user does not silently acquire a second, permission-less account.

**Access modes:**
- **Application API**: OAuth 2.1 **client-credentials** (or a long-lived API key as a simpler option), scoped by the application's RBAC permissions.
- **User tokens**: OAuth 2.1 authorization-code + PKCE, or a personal access token; both scoped to that user's RBAC permissions.

### Local password authentication
The baseline credential, present whether or not any federation is configured.
- **Hashing**: **Argon2id**, per OWASP's current guidance (minimum 19 MiB memory, 2 iterations, parallelism 1). Never plaintext, never a fast general-purpose hash.
- **Policy**: minimum 12 characters; long passphrases permitted (up to at least 64 characters) so length is not fought with composition rules; a blocklist check against known-breached passwords.
- **Change**: requires the current password as well as the new one, and **invalidates other sessions** on success — a changed password that leaves old sessions alive is not a change.
- **Reset**: single-use, expiring, emailed token. Two properties that are security requirements rather than features: the reset flow **cannot be used to lock a user out** (it never disables the account), and a successful reset invalidates existing sessions.
- **Login protection**: rate limiting and progressive back-off on failed attempts, with **audit logging of every failure**. A reset path that an attacker can trigger to deny a user access is an availability bug, so lockout is bounded and always leaves a recovery route.
- **Not a weaker path**: local-password sessions carry the same claims, the same expiry handling, and the same CSRF protection as any other authenticated session (see Security & Data Protection).

## Data Privacy & Consent
Vaah provides the privacy mechanisms in core; **the operator of a self-hosted deployment is the data controller** and establishes the lawful basis for their own processing.
- **GDPR-ready**: Vaah provides working consent capture and storage, a data-deletion workflow reaching core and module data through the shared service layer, retention configuration, and data export. This is a statement about the mechanisms Vaah ships — not a claim that Vaah determines lawful basis for a given deployment. **The one documented limit**: erasure does not reach the audit log, which is purged on its own configurable retention period instead.
- **Consent management** is a core capability: a consent record per subject, versioned consent text, withdrawal at least as easy as grant, and consent **enforced as a gate** on processing rather than merely recorded.
- **Data subject rights**: access, export, rectification, and deletion as a documented workflow across core and module data. **Erasure hard-deletes** — including records that were soft-deleted — and is not recoverable; the platform's recoverable soft delete governs ordinary business deletes only, and the restore affordance is hidden for erased records.
- **Data residency**: not Vaah's responsibility. Under the single-tenant self-hosted model, one deployment and one database sit wherever the operator placed them, and Vaah does not phone home (see Deployment model, Deployment & Self-hosting).
- **Sub-processors & AI providers**: not Vaah's responsibility. Because LLM keys are bring-your-own, the provider relationship and its data-processing terms belong to the operator; Vaah ships no first-party model path and processes no data on anyone's behalf. *If a first-party or proxied model path is ever added, this changes.*
- **Audit log scope**: personal data in the audit log is subject to a configurable retention period applied independently of the erasure workflow, and the audit log is documented as outside that workflow (see Audit Logs).

## Security & Data Protection
Security and data protection are enforced across the whole platform — core and modules alike — with secure defaults.
- **Encryption**:
  - **In transit**: TLS for all traffic (web, API, and internal service calls), with HSTS.
  - **At rest**: encryption of databases and file/object storage; field-level encryption for sensitive/PII fields where required.
- **Secrets & key management**:
  - Secrets (DB, SMTP, storage, and the BYO LLM key) are stored encrypted (a managed secret store / KMS or an encrypted config layer) — never in plaintext in code or the UI.
  - Rotation without downtime where feasible; secrets are injected into the service layer only, never exposed to the UI or agents (agents/workflows never hold infrastructure credentials — see Agentic Platform).
- **Security hardening**:
  - Security headers (CSP, HSTS, `X-Content-Type-Options`, frame-ancestors), a strict CORS policy, and rate limiting (see API Design Conventions).
  - **CSRF protection on every cookie-authenticated state-changing request** (`POST` / `PUT` / `PATCH` / `DELETE`). Session cookies are the primary browser credential (required by SSE — see D-4), which makes this a real surface, not a formality: a cross-origin `POST` to a mutating endpoint would otherwise carry the victim's session automatically. Defence is layered — **`SameSite` cookies**, an **anti-CSRF token** issued per session and required on mutation, and a **strict CORS policy** as the outer boundary.
  - **Where it does not apply, deliberately**: requests authenticated by a bearer token (OAuth access token, personal access token, API key) are not CSRF-vulnerable, because the browser does not attach the credential automatically. Requiring a CSRF token there would break every non-browser client — MCP hosts, external agents, CLI tools — for no security gain. *This is why the rule is stated as "cookie-authenticated", not "all authenticated": a blanket CSRF requirement is either ignored by API clients or bypassed where it matters.*
  - Dependency & SCA (software composition analysis) vulnerability scanning in CI; known-vulnerable dependencies are flagged.
  - Safe defaults: least-privilege, fail-closed access, and no debug endpoints in production.
- **Session & token management**: session timeouts, concurrent-session control, and token revocation (see API Authentication).
- **Module responsibility**: modules inherit the platform's security model (RBAC, encryption, secrets, audit) and cannot bypass it.

## Observability
Platform-level observability (distinct from agent observability in Agent Operations) for debugging and running the platform in production.
- **Logging**:
  - Structured application + error logs with levels (debug/info/warn/error) and a configurable log level.
  - A logs view in the dashboard to inspect and filter error and application logs (by level, time, request, user, module).
  - Correlation: every request carries a correlation/trace ID so a request can be followed across the service layer; agent and workflow actions reuse the same trace (links to agent/workflow traces).
  - **OpenTelemetry** as the tracing standard — correlation IDs are its trace/span identifiers, not a parallel mechanism. One agent action is a single trace spanning the model call, the tool call, and the resulting database writes.
  - **SLOs and error budgets** defined per capability as stated targets.
  - Error capture: unhandled exceptions with stack traces and context, surfaced for debugging.
  - Modules emit logs through the same facility.
- **Monitoring**:
  - A status/monitoring view: overall system health, per-component health, and scheduled jobs (cron/workflows) — next run, last run, success/failure, duration.
  - Dependency monitoring: continuous checks on infrastructure — DB connectivity, cache (e.g. Redis), message queue, media storage, and external services — with status history and alerts on degradation. Under the single-tenant deployment model there is no per-tenant dimension to these checks.
  - Background task visibility: running/completed/failed jobs and queues.
  - Alerts: on health degradation or job failure, delivered via the platform's notification/email system.
- **Health**:
  - Public (unauthenticated) health endpoint(s) for uptime/monitoring (basic liveness).
  - A detailed, authenticated health check reporting dependency status (DB, cache, queues, external services).

## Performance
"Highly performant" is a top priority; these are the concrete requirements and mechanisms:
- **Targets**: fast, responsive interactions — sub-second list/detail loads and a snappy dashboard; API time-to-first-byte targets under normal load.
- **Data access**: DB indexing on hot query paths, cursor-based pagination for large lists, N+1 avoidance, and query review for scaffolded CRUD.
- **Caching**: Redis for hot data, sessions, and queues, plus a CDN for static assets and media; cache invalidation tied to record changes (see Data & Storage).
- **Frontend**: code-splitting and lazy-loaded routes, list virtualization for large tables, efficient re-renders, and TanStack Query caching/deduplication.
- **Real-time**: efficient WebSocket/SSE fan-out — only the relevant updates reach the relevant users, not a broadcast to everyone.
- **Scalability**: a stateless API tier that scales horizontally, DB connection pooling, and queue-based background processing for heavy work (monitored via Observability).
- **Metrics**: performance metrics (latency, error rate, saturation) surfaced in Monitoring.

## Deployment & Self-hosting
Self-hosting is the primary, first-class deployment model: an organization runs Vaah on its own infrastructure (VPS, VM, private cloud, on-prem, or container orchestrator). A managed offering may exist later, but self-host is the priority and the one that must always work.
- **Packaging**: containerized (Docker / Docker Compose) as the default install path, plus bare-metal/VM (Node.js) and Kubernetes for scale; a documented, reproducible install.
- **Configuration**: configured via environment variables / config (app URL, DB connection, storage backend, LLM provider + key, secret, …); the onboarding wizard handles first-run setup, ongoing config is env-based.
- **Dependencies**: Node.js (NestJS), a relational database (e.g. PostgreSQL), Redis (cache + queue), and a storage backend (local or S3, see Data & Storage); an LLM provider (BYO key). Minimum requirements documented.
- **Self-contained core**: the core is fully functional self-hosted without phoning home to a central Vaah service. Remote calls are limited to optional items (update/upgrade checks, premium/SaaS module license validation).
- **Upgrades**: self-service upgrade path — pull the new version, run migrations/seeds, restart; documented and reversible (see Settings → Upgrade).
- **Data sovereignty**: all data (DB, files) stays on the user's infrastructure; exports and backups are available (see Platform).

## Configuration & Feature Flags
Centralized, environment-aware configuration plus feature flags for controlled rollout.
- **Configuration**:
  - Deployment-time config via environment variables / config (see Deployment & Self-hosting), validated against a typed config schema.
  - A runtime config surface in the admin console for settings that change without redeploy (see Settings).
- **Feature flags**:
  - Enable/disable features (core or module) without redeploy — scoped per deployment or per user/role.
  - Gradual rollout: roll a feature out to a subset of users first, then broadly.
  - Flags are visible (which are on, where) and can be toggled from the admin console or API.
- **Module gating**: flags can gate a module's features or its activation, complementing Licensing & Distribution entitlements.
- **Safe rollout**: a flag change is reversible; where destructive it requires confirmation; changes are audit-logged.

## Disaster Recovery
The platform must survive failures and be recoverable, with defined recovery objectives.
- **High availability**: a stateless app tier that scales horizontally (see Performance); database and cache offer HA options (replication / failover) appropriate to the deployment.
- **Backups**: scheduled full + incremental backups of databases and storage (see Platform), off-site / cross-region where required.
- **Recovery objectives**: defined RPO (maximum data loss) and RTO (maximum recovery time) targets, set per deployment.
- **Backup and restore are out-of-process operations — the CLI is the primary interface, not the admin panel.** Both must be creatable and recoverable through a command-line operation against a deployment that is not running, working on the database and storage directly rather than through the application.
  - *The reason is the failure being designed for:* the moment a backup matters most is an instance that cannot start — corrupted migrations, a bad upgrade, a module that bricked the app. A restore that requires a healthy running application fails at exactly the moment it is called for, and a backup that can only be verified through the app is a hope rather than a control. Comparable platforms have shipped in-app restore and had it fail for precisely this reason.
  - **Consequence: the admin panel's backup and restore screens are conveniences over the same CLI**, not the source of truth. Anything the UI offers, the CLI must offer — no capability that exists *only* in the UI. A recovery path reachable only by an authenticated browser session is not a recovery path.
  - **Backup before restore, always.** The CLI takes an automatic pre-restore backup unless explicitly told otherwise, because a restore is itself a destructive, hard-to-reverse act on data that may have changed since the backup was taken.
  - **A restore is verified by drill, not by assertion.** "The backup ran successfully" is a log line; it is not evidence that the data can be read back.
- **Restore**: tested, documented restore procedures (point-in-time where supported) — a first-class operation, not an ad-hoc script.
- **Failover & graceful degradation**: clear failover behavior for DB, cache, and queues; the platform degrades gracefully (queues buffer, cache misses fall through to the DB) rather than failing hard.
- **Migration safety**: upgrades and schema migrations are guarded (backup + dry-run) and reversible where possible to prevent data loss (ties to Deployment & Self-hosting upgrades). **Expand/contract is mandatory** — a breaking schema change never ships in one release: the new shape is added alongside the old (expand), and the old one removed in a later release (contract). *A contract step therefore has no down-path — rolling back means rolling forward. This is declared in the migration itself, not discovered during an incident.*
- **DR testing**: periodic restore / failover drills to validate RPO/RTO. **The drill restores via the CLI on a machine where the application has never successfully run** — a drill performed inside the running admin panel proves the happy path is intact and nothing else. A drill that has never been executed against a stopped instance is an untested assumption.

## Resilience & Failure Handling
Behaviour under failure inside a running request — the complement to Disaster Recovery, which covers recovery after a failure.
- **Timeouts on every outbound call**, no exceptions — database, cache, queue, LLM providers, outbound webhooks, external services. A hung dependency must not hold a request open indefinitely.
- **No unbounded work inside a database transaction**: calls to external services (notably LLM providers) happen *outside* transactions, which open only around the final write. An AI provider that stalls must not hold a database connection and row locks — under concurrent agents this exhausts the pool and takes the whole platform down.
- **Retry policy is declared per job, not automatic** (modelled on the Laravel job API — each job states its own attempts, backoff, and timeout). Retries use exponential backoff with jitter. No retry unless the job asks for it.
- **Idempotency**: jobs with outside effects (email, payment APIs, webhooks) are the author's responsibility to make idempotent; the platform provides idempotency keys and a transactional outbox as the supported mechanism.
- **Dead-letter handling**: a job that fails and was not eligible for retry is still surfaced, with an explicit note that it was not retried and why — nothing fails silently.
- **Circuit breakers on external services only** — LLM providers, payment gateways, outbound webhooks — never on the database, where stopping would deepen an outage. Cache reads may break to the database, matching the graceful-degradation guarantee above.
- **Distributed locking** for scheduled jobs and single-flight work, so a horizontally-scaled tier cannot run the same scheduled task twice; locks carry a bounded expiry so a crashed holder cannot deadlock the schedule.
- **Multi-step operations use one database transaction, not a distributed saga.** Vaah is single-tenant on one database, so compensation patterns are unnecessary.
- **Capacity protection**: a bounded worker pool per dependency type, so one failing external service cannot consume all platform capacity.
- **Timeout tiers**: short timeouts for ordinary requests; a separate, **configurable** longer ceiling for LLM/agent calls (**default 5 minutes**), so interactive pages fail fast while long-running agent work is not cut off. *The split between tiers matters more than either number.*

## Accessibility
Vaah targets **WCAG 2.2 level AA**. Accessibility is a procurement gate in enterprise deals — asked in security questionnaires and vendor reviews — so it is specified here rather than left as a UI preference.
- **Contrast enforced as design tokens**: colour tokens carry contrast constraints, so a module overriding a token (see Design Tokens & Theming) cannot produce unreadable text. This is what keeps contrast intact across third-party modules.
- **Keyboard and focus are platform conventions**, not per-module choices: tab order, focus trapping in modals, focus restoration on close, and skip-to-content are platform rules that modules must follow. Per-module focus behaviour yields an inconsistent assembled UI.
- **Automated scanning in CI is mandatory, and is a floor not a ceiling**: scanners catch roughly 30–40% of WCAG issues. Screen-reader semantics, focus management, keyboard traps, error announcement, and custom-component contrast are the remainder, owned by design review and manual testing.
- **Reduced-motion supported**, and reduced-transparency where applicable.
- **Modules are not gated**: Vaah does not block or flag a module that fails accessibility conformance — consistent with its position on module conformance generally. The module manifest carries an *optional* accessibility conformance field so operators can see it, but enforcement is not Vaah's responsibility.

## Module trust & the marketplace
Vaah follows the **WordPress plugin model**: anyone can build and install a module; the core's responsibility is to confirm it is *configured correctly as a module*, then step aside.
- **Vaah verifies registration, not trustworthiness.** The sanity checks listed under Modules, run at activation, dry-run first, failing closed.
- **No signing, no author verification, no sandboxing, no quality gating.** Modules run in-process with the application's database access and privileges. *This is a deliberate trade: modules reuse the shared service layer precisely so RBAC, audit, and attribution apply automatically, and isolation would mean reimplementing that layer across a process boundary. **Vaah buys uniform governance for modules by giving up isolation from them.** A module's migrations run with platform credentials and are not scope-checked — the same trust boundary as its runtime code.*
- **Marketplace curation: inform, not enforce.** Listings publish what the author declares.
- **Accepted limitation**: a module can pass every sanity check and be malicious or broken. The operator assumes this by installing. **Declared capabilities are shown before install** so that "yes" is an informed decision rather than a blind one — and contributing an **agent** is surfaced separately, because a module that introduces an autonomous actor with scoped tools is asking for the highest-value capability in the system, and nothing bounds a person's willingness to click "yes" to a listing that looks reasonable.
- **Behavioural verification is the module author's responsibility**, run in the module's own CI. Vaah provides no central compatibility test farm.

## Event bus
One mechanism carries workflow triggers and real-time updates. It is a **documented extension point** — modules both publish and subscribe.
- **Transport: Redis pub/sub**, matching the real-time transport (D-4) and the queue (O-6) — one Redis dependency rather than two, and it survives restarts so an event can still fire a workflow after one.
- **Two delivery tiers**, matched to what each consumer needs:
  - **At-most-once for in-process subscribers** (UI live updates) — a dropped refresh is self-correcting via the next event or poll.
  - **Durable via the transactional outbox** for anything that must not be lost — a workflow trigger or an email is written in the same transaction as the business change, then published.
- **Events are namespaced and typed**, so module events cannot collide with core events and subscribers can filter.
- **Module subscriptions are declarative**, making a module's event surface visible and auditable rather than buried in code.

## Frontend module loading
How a separately-built module's UI reaches the admin panel — a React + Vite SPA.
- **Modules ship runtime-loadable bundles** and register routes, menu items, and components at activation. Installing a module is a data operation, not a rebuild — this is what makes the module marketplace viable, since build-time bundling would turn every install into a redeployment.
- **The host supplies shared dependencies** — React, design tokens, and the query client. A module must not ship its own copy; duplicate React instances are the standard failure mode of this pattern.
- **Modules consume core design tokens** (see Design Tokens & Theming), so a theme change applies without a rebuild.
- **Version compatibility is checked before load**, using the module's declared `vaah_compatibility` range.
- **A module whose UI fails to load does not break the panel** — it is skipped and disabled with a logged error and an admin flash message, matching the module failure-containment policy.
- **CSS is shared with the host for v1**, favouring a consistent assembled UI. Style isolation is a later hardening step, not a v1 requirement.

### Delivery mechanism
- **A module ships an ESM bundle the admin imports at runtime.** Because React 19 publishes no UMD builds, the historical global-`window.React` contract is unavailable; the equivalent today is an **import map** published by the host, through which a module resolves `react`, `react-dom` and `vaah/ui` as singletons. A module author writes ordinary imports and receives the same React instance and the same components core does.
- **Externals are an obligation, not an optimisation.** Duplicate React breaks hooks and context **even when both copies are the identical version** — two dispatcher slots mean no context propagation regardless of version match. Inlining a host-provided library is the single most common cause of failure in this architecture, and it has been measured at around 99% of runtime-bundle plugin faults in comparable ecosystems. A module that inlines `react`, `react-dom` or `vaah/ui` is misconfigured, and should fail its pre-activation checks.
- **Isolation is by capability, not by realm.** A module receives a declared surface — it can render inside slots and call core's published services, and it cannot reach the host router, global stores or internals. It does **not** run in a sandboxed frame.
  - *Why not a frame:* a frame cannot use Vaah's own components, so every module would reimplement tables, forms and dialogs and the assembled panel would stop looking assembled. It would also force current-record, selection, theme and permission state across a message boundary — and a message bridge between two origins is itself a disclosed cross-site-scripting surface in comparable platforms. Frame isolation suits embedding third-party *content*, which is not what module UI is.
  - *The honest cost:* a module that loads and then misbehaves at runtime is **not** contained — there is no realm boundary to stop it. Load *failure* is contained (skipped and disabled, as above); load *success* followed by a fault is not. This follows from the trust model (see Module trust & the marketplace): Vaah gives uniform governance for modules by choosing not to isolate them from the panel.
- **Every module-rendered surface is attributed in the UI**, so a fault can be traced to Vaah or to a named module without guesswork. The trust model makes this a usability requirement, not a nicety: an operator who cannot tell which module broke cannot act rationally.

### The module development loop
- **A module developer runs `npm run dev` in the module and never builds Vaah's admin.** The admin dev server proxies `/modules/<name>/ui` to that module's dev server, so module code loads through exactly the same path it will use in production. This is the constraint that keeps the whole architecture honest — if enabling a module ever required rebuilding the admin, activation would stop being a data operation and the marketplace model would collapse.
- **Single origin, so no CORS.** Proxying rather than loading from the module's own origin keeps module bundles same-origin with the admin, which is also what keeps a strict `Content-Security-Policy` workable — under `strict-dynamic`, browsers ignore `https:` allowlists, so a bundle fetched from another origin simply cannot be loaded.
- **Version compatibility is checked before load** against the declared range, and the same check runs in development, so a module author learns about incompatibility immediately rather than at install time on someone else's deployment.
- **Expect rebuild-and-reload, not hot module replacement.** Bundling a module as a library does not give the fast-refresh loop a web application gives. This is documented as a property of the model rather than left to be discovered as a missing feature.
- **Host-supplied singletons mean a misconfigured module can degrade the whole panel**, so the pre-activation checks treat an inlined host dependency as a fault, not a warning (see Modules → sanity checks).

## UI Extension Slots
How a module places its UI *inside* an existing Vaah screen, rather than only on its own pages. Adding menu items and pages is not the same as extending a core screen, and without a rule the assembled panel has no defined shape.
- **Slots are named, versioned, and declared by core.** A module fills a slot; **only a core change can create one**, so a module can never inject into an undeclared location. Core owns the surrounding layout and behaviour — the tab strip, the table structure, the action-bar placement — and the module owns only the content inside the slot.
- **The slot catalogue is closed and documented.** Two categories:
  - *Adding a surface* (already decided): dashboard widget, left-menu item, top-menu item, profile-dropdown item, Settings section.
  - *Extending a core screen*: a **tab on a record detail page**, a **column in a list/table view**, an **action button on a record page**, and a **related-items panel on a record page**.
- **A module declares its slot fills** with the target screen, the slot, a label, and a **render condition** (e.g. "show this tab only when the record is a User"). Conditions are evaluated by core, so a module does not decide whether it belongs on a given record.
- **A missing slot degrades that piece only.** If a module targets a slot that does not exist in the running Vaah version, that slot fill is skipped with a logged warning and the module's own pages still load. A module built against a newer core is not dead on arrival in an older deployment — this is what makes `vaah_compatibility` a range rather than a pin.
- **Ordering is module-declared.** Each fill carries a **sort weight**; core orders by weight, then by module name as a deterministic tie-break. A module author controls placement without depending on install order, and two deployments that installed the same modules in different orders present identically.
- **Slot content is permission-gated and attributed like any other UI.** A fill is only shown to a user who holds the module's permissions, and agent-driven activity originating in a slot is attributed in the usual way. A module cannot surface data to a user who would be refused it through the API.
- **Slot content runs in the admin panel's origin**, so it holds the same power as the logged-in user. This follows directly from the trust model (see Module trust & the marketplace) rather than being an additional grant: Vaah gives uniform governance for modules by giving up isolation from them.
- **Accessibility conventions are the platform's, not the module's** — a module-supplied tab or panel inherits core's focus management and keyboard handling rather than reimplementing it (see Accessibility).

## Content Sanitization
Rich-text and HTML handling is a **security control**, not a formatting convenience: both users and agents produce content, and a self-hosted deployment gives one stored XSS full access to that instance.
- **Sanitized on write** at the platform's rich-text service, so every reader — UI, API consumers, exports — is safe by default.
- **Sanitized again on render**, as defence in depth for content arriving via migration, import, or seeds.
- **Allowlist-based**: an explicit set of permitted tags and attributes. Denylists fail open on novel payloads; an allowlist fails closed.
- **URL and link safety**: `javascript:` and `data:` URIs blocked; `target="_blank"` forced to `rel="noopener noreferrer"`.
- **Agent-generated content follows the identical path** as user content — there is no privileged route for agents to write unsanitised HTML.
- **A core capability that modules consume**, consistent with media and taxonomy.

## Delivery & Release Engineering
How changes reach a deployment, and what must be true before one does.
- **Trunk-based development** with short-lived branches; the spec-driven flow produces auditable commits.
- **Required checks on every change**: lint, type-check, unit and integration tests, migration validation (up and down where a down-path exists), expand/contract enforcement, dependency SCA, OpenAPI backward-compatibility, and the accessibility scan floor.
- **Expand/contract migrations are mandatory** for core and for CLI-generated migrations — see Disaster Recovery.
- **Supply chain**: an SBOM per release, signed artifacts, and provenance attestation. The module lifecycle loads third-party code at runtime, so artifact provenance is a security control rather than a formality.
- **Semantic versioning on the contract boundary**: the major version is the compatibility boundary that modules declare against (`vaah_compatibility`); minor and patch releases never break the extension contract.
- **Support window**: the current and previous major receive fixes, with a defined security-fix window for the previous major.

## Developer documentation
Docs are part of the product, not a byproduct of it — the module ecosystem cannot start before the docs exist. They are built in layers, each on its own clock:
- **Conceptual layer (the model): written as decisions are made.** What a module is, the declaration model, the four outputs, the module lifecycle, slots, and the trust model — lifted from the requirements as each decision settles. This layer rarely changes, and it is the contract the build itself runs against.
- **Contract reference: written at contract freeze, kept generated.** The declaration schema, manifest fields, output formats, and core service APIs are documented the way OpenAPI is — generated from the source of truth, so the reference cannot drift from what the platform actually accepts.
- **Tutorials: drafted during the build, verified against the running product.** Every page carries a "verified against vX.Y" stamp, re-verified on each release.
- **Reference module: built alongside the runtime.** Core dogfoods the loader and is the first example; a small, complete third-party-style module exercises every mechanism (declaration, migrations, slots, permissions, MCP tools) and is the primary teaching artifact.
- **Machine-consumable**: the docs have a second reader — coding agents that author modules. They must be complete, structured, versioned, and free of "see the source code" escapes. The declaration is the single source of truth for the four outputs, and for the module-authoring reference as well — docs as a fifth output.
- **Quality bar**: the publishable requirements — full API coverage, an example per feature, freshness SLA — live in Platform Standing & Adoption.

## Platform Standing & Adoption
Beyond building the platform, Vaah must *earn* its standing: be the developers' favorite choice, and a platform that AI assistants (Claude, ChatGPT, Gemini, …) confidently recommend. These are the evidence, standards, and quality bars to maintain.

### Quality bar
The measurable, publishable bar that makes the platform trustworthy and elite:
- **DORA metrics** — target *elite* on all four: Deployment Frequency, Lead Time for Changes, Change Failure Rate, and Failed-Deployment Recovery Time (small batches, trunk-based dev, CI/CD, decoupled architecture, fast feedback).
- **SLOs / reliability** — uptime ≥ 99.9%, p95 API latency targets, MTTR, and graceful degradation; a public status page.
- **Security posture** — OWASP ASVS (L2), SOC 2 Type II, dependency SCA in CI, and a transparent, public security posture (see Security & Data Protection).
- **Stability** — semver, backward-compatible changes, explicit deprecation windows; no unexpected breaking changes (counters "obsolescence," a top developer rejection driver).
- **Documentation quality** — 100% of the public API documented, an example for every feature, a freshness SLA, and full-text search.
- **Community SLAs** — target median first-response time on issues, active contributor count, and triage turnaround.

### Developer adoption
How the platform becomes the developers' favorite (the "favorite" bar is largely about *not* triggering the top rejection drivers — security/privacy, pricing, API quality, usability, obsolescence, weak AI capability):
- **DX excellence** — fast time-to-first-result, excellent docs, low-friction onboarding, fast local dev, and clear error messages (see Developer Experience).
- **Published evidence (social proof)** — public benchmarks, customer case studies, a "used by" logo wall, GitHub stars/downloads, and a public roadmap.
- **Community & ecosystem** — responsive maintainers, a welcoming contributor path, and a rich set of examples, templates, and modules (see Modules, Extension Points).
- **Standards alignment** — OpenAPI 3.0.x, MCP, A2A, OAuth 2.1/OIDC, RFC 9457, JSON:API, WCAG 2.2, and semver (see API Design Conventions, Agent-Native Platform).
- **Open-source core** — no prohibitive pricing or lock-in; commercial value lives in modules (see Licensing & Distribution).

### AI discoverability & recommendation
How the platform becomes a confident recommendation from LLMs — which weigh (a) what they already know, (b) what they can find now, and (c) whether they can use it:
- **Training-data presence** — wide adoption and active discussion (GitHub, Stack Overflow, tutorials, YouTube), a consistent brand name, presence in authoritative references (Wikipedia, MDN, "best-of" lists, comparisons), and positive sentiment.
- **Answer-engine visibility (GEO/AEO)** — the "SEO for AI": a `/llms.txt` map (see Agent-Native Platform), question-form direct-answer content for the queries developers actually ask, structured data (schema.org), primary-source docs, and a crawlable site that does not block AI crawlers.
- **Agent-usability (the differentiator)** — an LLM favors a platform it can *drive*: the Vaah MCP server, the A2A Agent Card, and a clean OpenAPI contract (see Agent-Native Platform, API Design Conventions).
- **Citation monitoring** — track where/how often the platform is cited by AI (e.g. Bing "AI Performance" and Google "Search Generative AI performance" reports, plus LLM citation tracking).

### Positioning
A single, clear statement of what Vaah is the default for — e.g. "the open-source, agent-first platform for building business software" — used consistently across docs, site, and the agent self-description layer so both humans and LLMs form one stable mental model.

## Tech Stack
- NestJs as Backend and API
- React + Vite SPA as Frontend
- Shadcn/ui as frontend component library
- TanStack Query (server/real-time) - Zustand (global client + the frontend module registry, where runtime-loaded module bundles register routes, menu items, and widgets).
- Redis pub/sub as the event bus carrying workflow triggers and real-time updates (see Event bus).

## Terminologies
A glossary of the key terms used throughout this document, grouped by area. It covers both the **industry-standard terms** the platform is built on (e.g. MCP, A2A, RBAC, OAuth 2.1, cursor pagination, RPO/RTO) and **Vaah-specific terms** (e.g. dual attribution, detach on edit, the frontend module registry) — the latter are defined here so all readers share one vocabulary.

### Architecture & platform
- **HMVC (Hierarchical Model-View-Controller)**: a modular architecture where the core and each module have their own MVC (model-view-controller) structure. The hierarchy is **one level** — a module may contain sub-modules, but there is no recursion. Nesting is organisational (grouping files and routes), not inheritance from a core base.
- **Modular monolith**: a single deployable application composed of strongly-bounded internal modules — the overall style of Vaah.
- **Core**: the open-source platform foundation that modules install into and extend.
- **Deployment**: a self-hosted Vaah instance serving one business/organization; the unit of isolation and the unit of scaling (see Deployment model).
- **Service layer**: the shared NestJS layer every caller (UI, API, agent, workflow) goes through; it enforces RBAC, audit, and attribution (see Agentic Platform → Security boundary).
- **Extension point**: a stable, documented surface where a module plugs into the core without touching internals (see Extension Points).
- **Self-hosting**: running Vaah on your own infrastructure — the primary deployment model (see Deployment & Self-hosting).
- **Cross-module service access**: reaching data a module does not own through core's published services and providers instead of touching another module's tables, so RBAC, audit, and attribution still apply (see Extension Points → Cross-module service access).
- **Module declaration**: a module's self-describing manifest — its routes, entities, permissions, slot fills, events, jobs and compatibility range — from which HTTP routes, OpenAPI schema, agent tools and client shapes are all generated. **Declaration is data, not code**, so the admin can read it over the API without importing a backend class (see Module Architecture).
- **Route namespacing**: every module endpoint lives under `/modules/<name>/`, so the origin of a route is unambiguous in logs, audit records and the contract.
- **Capability isolation**: containing a module by limiting what it can reach — a declared surface of services, slots and tokens — rather than by running it in a separate browsing realm. Isolation by permission, not by frame (see Frontend module loading).
- **Host-provided singleton**: a shared dependency (React, the design system, the query client) supplied to modules by the host so every participant resolves the *same instance*. Duplicate React instances break context propagation even at identical versions.

### Modules & extension
- **Module**: an installable unit (e.g. cms, crm, epr) that extends the core; it ships its own MVC, migrations, seeds, UI, permissions, and MCP tools.
- **Frontend module registry**: the UI-side registry where runtime-loaded module bundles register their top/left menu items, routes, and dashboard widgets (see Frontend module loading).
- **UI extension slot**: a named location on a Vaah screen that core declares and a module may fill — a record-detail tab, a list-view column, a record-page action button, or a related-items panel. The catalogue is closed: only a core change creates a slot, so a module can never inject where core has not provided a place (see UI Extension Slots).
- **Slot fill**: a module's declaration of what it renders into a slot — the target screen, the slot, a label, a render condition, and a sort weight. Ordering is by weight, then module name.
- **Sort weight**: a module-declared integer used to place a slot fill or menu item, so ordering does not depend on the order modules happened to be installed in.
- **Module registry / marketplace (distribution)**: the vaah.ai registry for publishing, discovering, and installing modules — a later-stage feature (see Modules).
- **Required seeds / Sample seeds**: two seed categories — required seeds must run on activation; sample seeds load demo data.
- **Detach on edit**: when a user edits a module-seeded workflow or agent, it becomes user-owned and no longer follows the module's lifecycle.
- **Entitlement**: a module's licensed access — feature/seat caps for freemium, a verified license for premium, remote auth for SaaS.
- **License types (OSS / freemium / premium / SaaS)**: the four ways a module can be licensed; the core runs them side by side (see Licensing & Distribution).

### Agents & AI
- **Agent**: a first-class actor (distinct from a human user) that performs operations, with its own identity, scoped credentials, and audit trail.
- **Agent actor**: agents modeled as a distinct actor type in RBAC, able to hold delegated authority.
- **Runtime agent**: the built-in chat agent that operates the live app (CRUD/domain ops + UI rendering).
- **Build-time agent**: coding agents (e.g. Claude Code, Copilot) that scaffold and extend modules on the platform.
- **Sub-agent**: an agent that can be delegated to by another agent/orchestrator.
- **MCP (Model Context Protocol)**: the open standard for exposing tools, resources, and prompts to LLMs; Vaah exposes an MCP server. The current core is **stateless** — auth and context travel as per-request headers, with no protocol session ID.
- **MCP Apps**: an official MCP **extension** (not core) — components rendered as sandboxed-iframe HTML apps inside any MCP host via a `ui://` resource. The Vaah MCP server is fully functional without it; the tool result is the contract, the rendered app the enhancement.
- **MCP resource server**: the OAuth 2.1 role a Vaah MCP server plays — an authenticated resource whose capabilities are exposed to a client acting under a scoped identity.
- **A2A (agent-to-agent)**: the protocol for one agent delegating a whole task to another; Vaah exposes an **Agent Card** (a JSON manifest of capabilities and connection info).
- **BYO LLM**: bring-your-own LLM provider + key (no core metering/billing).
- **Guardrails**: input/output/action safeguards on the model layer — prompt-injection defense, PII redaction, output validation (see Model Management & Guardrails).
- **Dual attribution**: recording both the accountable user principal (`created_by`/`updated_by`/`deleted_by`) and the agent instrument (`is_agent` + `agent_identity`) on every agent-made change.

### Audit & observability
- **Fail open (audit)**: when an audit entry cannot be written, the business operation proceeds and the failure is logged and alerted.
- **Correlation/trace ID**: the identifier following one request or agent action across the service layer, agents and workflows — carried as the OpenTelemetry trace/span ID.

### Data & CRUD
- **CRUD scaffolding**: auto-generating an entity's endpoints, UI, RBAC, migration, and MCP tools in one step.
- **Soft delete**: deletion via `deleted_at` (recoverable), not a hard delete.
- **Migration**: a versioned schema change applied when a module is activated or upgraded.

### API
- **REST**: the canonical, versioned read/write API contract (`/v1/...`), served alongside MCP tools.
- **OpenAPI 3.0.x**: the machine-readable API description — the single source of truth for codegen, docs, and MCP.
- **RFC 9457 Problem Details**: the standardized HTTP error format (`application/problem+json`).
- **Cursor pagination**: stable, token-based list pagination (`page_token` / `next_page_token`).
- **JSON:API**: the resource-shape conventions (`data`/`links`/`included`, resource objects) adopted for a uniform API envelope.

### Security & auth
- **RBAC (role-based access control)**: access control via roles → permissions → users; verified at the code level before every operation.
- **Field-level permissions**: read/write control at the column/field level, not just the entity level. *Not enforced on the reporting surface, which is superadmin-only — an arbitrary user-composed query cannot be field-checked at runtime (see Reports).*
- **OAuth 2.1 / OIDC**: the authentication standard — client-credentials for applications, authorization-code + PKCE for users.
- **PAT (personal access token)**: a long-lived, scoped user token.
- **Idempotency key**: a token that lets a create/update be retried safely without duplicates.
- **Security boundary**: agents and workflows never hold infrastructure credentials; all actions run through the shared service layer.
- **Local password authentication**: the always-available username/password credential — Argon2id hashing, minimum 12-character policy with breached-password blocklisting, current-password-required change, single-use expiring reset, and rate-limited login. The baseline method when no federation is configured.
- **Federated identity**: sign-in delegated to an external provider (OIDC, SAML, social). Opt-in — offered only when configured via Settings or environment variables, and resolving to the same internal principal, RBAC, and audit trail as any other sign-in.
- **CSRF (cross-site request forgery)**: an attack that induces a browser to send a state-changing request carrying the victim's session cookie. Mitigated by SameSite cookies, a per-session anti-CSRF token on mutations, and strict CORS. **Applies only to cookie-authenticated requests** — bearer-token clients do not attach credentials automatically and are unaffected.
- **Cascade on user deletion**: when a user is deleted, a `user.deleted` event fires on the event bus and each module applies its own soft- or hard-delete policy to the rows referencing that user. Hard delete previews the affected counts per module before destroying anything.

### Workflows & automation
- **Workflow**: a deterministic, fixed-path automation (trigger → steps → actions) with no LLM in the execution loop.
- **Trigger**: what starts a workflow — event, schedule, manual, or API/MCP.
- **Run-as identity**: the identity a workflow runs under (event-driven defaults to the triggering user; scheduled uses a system identity).
- **Visual builder**: the Zapier-style, top-to-bottom step editor for building workflows.

### Resilience
- **Timeout tier**: the per-call time limit before a dependency is abandoned — short for ordinary requests, a separate configurable ceiling for LLM/agent calls.
- **Circuit breaker**: a temporary halt on calling a failing external service, so it is not hammered while down.
- **Dead-letter**: the holding area for failed jobs, where they are inspected and may be retried.
- **Transactional outbox**: writing the job record and the business change in the same database transaction, so a job can never exist without its trigger (or vice versa).
- **Idempotent**: an operation safe to run more than once with the same result — the requirement for any retried job with outside effects.

### Privacy
- **Data controller**: the operator running a self-hosted deployment — not Vaah. Vaah ships mechanisms; the operator establishes lawful basis.
- **GDPR-ready**: Vaah provides consent capture and storage, retention configuration, data export, and a deletion workflow reaching core and module data. A statement about shipped mechanisms, not a per-deployment compliance determination.
- **Consent gate**: consent enforced as a precondition of processing, not merely recorded.
- **Data subject request (DSAR)**: a request from an individual to access, correct, export, or delete their data.
- **Sub-processor**: a third party that processes data on a controller's behalf. With BYO LLM keys, the operator's model provider is the operator's sub-processor — not Vaah's.
- **Email deliverability**: whether a message reaches the inbox rather than spam — SPF, DKIM, DMARC, and sender reputation. **Not Vaah's responsibility** under the self-hosted model; the operator owns the sending domain and its DNS records.
- **Suppression list**: addresses that hard-bounced or complained and are skipped rather than retried, protecting the deployment's sender reputation. Reversible by the operator.
- **Transient vs. permanent bounce**: a temporary failure (full mailbox, greylisting) that is retried within the job's declared policy, versus a permanent failure (nonexistent mailbox) that is not retried, is recorded, and surfaces as a dead-letter.

### Accessibility
- **WCAG 2.2 AA**: the accessibility conformance target for Vaah's UI.
- **Design token contrast constraint**: a colour token carrying a minimum contrast ratio, so overriding the token cannot break text legibility.
- **Focus management**: the platform convention for where keyboard focus goes on open, close, and navigation — modals trap focus while open and restore it on close.

### Concurrency & content
- **Optimistic concurrency / version conflict**: every mutable record carries a version; an update naming a stale version is refused rather than silently overwriting a newer change.
- **Bulk overwrite**: a bulk operation too large to conflict per-row, so it is explicitly marked as overwriting — recorded as such in the audit entry.
- **Bulk import**: the default import surface on every scaffolded CRUD — file upload, explicit column mapping, validation before commit, operator-chosen partial vs. all-or-nothing, natural-key matching so re-import updates rather than duplicates, run as an audited queued job.
- **UTC everywhere**: all timestamps are stored in UTC; conversion to a zone happens at the client edge for display, or explicitly for scheduling and report bucketing. One storage convention means timestamps are directly comparable and sort correctly as plain values.
- **Cross-module service access**: reading or writing data a module does not own through core's published services and providers, rather than querying another module's tables — so RBAC, audit, and attribution still apply. Declared in the module manifest; cross-module *writes* require an explicit grant on both sides.
- **Declared module dependency**: a module naming another module (or a core capability) that it requires, checked at activation by the sanity checks — so dependency integrity is verified rather than assumed.
- **Superadmin-only reporting**: the reporting query surface is restricted to superadmins because a user-composed query cannot be field-level-permission-checked at runtime. The trade is deliberate: reporting is powerful enough that granting it to an ordinary role would make field-level RBAC decorative.
- **Event bus**: the Redis-backed publish/subscribe mechanism carrying workflow triggers and real-time updates.
- **At-most-once delivery**: a subscriber may miss a message without retry — acceptable for self-correcting consumers like UI live updates.
- **Runtime-loadable module bundle**: a module's frontend shipped to be loaded into the running admin panel at activation, so installing a module requires no rebuild of Vaah.
- **Rich-text sanitization**: allowlist-based cleaning of user- and agent-generated content, applied on write and on render.

### Operations & deployment
- **Feature flag**: a toggle to enable/disable a feature without redeploy, scoped per deployment or per user.
- **Observability**: platform logging, monitoring, and health (distinct from agent observability).
- **RPO / RTO**: recovery point objective (max data loss) and recovery time objective (max recovery time).
- **Out-of-process recovery**: backup and restore performed by a CLI against the database and storage directly, without a running application. The primary recovery interface — because an instance that cannot start is precisely the instance that must be recoverable, and a restore path requiring a healthy app fails when it is called for (see Disaster Recovery).
- **Deployment model**: single-tenant by design — one deployment, one business, one database; scaling to many customers means running many deployments.
- **i18n / localization**: the core's UI string-translation mechanism, reused by modules.
- **UTC everywhere**: the storage convention — every timestamp is stored in UTC and converted only for display (client-side) or where a specific zone is semantically required (schedules, report buckets, send windows).
- **Golden task**: a fixed agent scenario with an expected outcome, used as a regression check.
- **Red team (evals)**: adversarial scenarios probing prompt injection, data exfiltration, and privilege escalation.
- **Invariant assertion**: verifying a property of the outcome (the right records changed, permissions did not escalate) rather than an exact output — the only sound assertion against non-deterministic output.
- **Expand/contract migration**: adding a new schema shape alongside the old one, removing the old one in a later release — the pattern that keeps upgrades reversible.
- **Contract window**: the period after a change is released during which a problem counts as a regression rather than an accepted new behaviour.
- **Sanity checks**: the registration-time validations Vaah runs when activating a module — well-formed and correctly declared, never a judgement about behaviour.
- **Idempotent seed**: a seed that declares how it identifies its records, so re-running it updates rather than duplicates.
