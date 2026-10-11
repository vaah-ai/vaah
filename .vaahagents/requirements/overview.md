# VaahApp

## Overview
VaahApp is going to be an agentic HMVC Software Application Development Platform for rapid software development.

VaahApp is going to be a core software which can have completely separate modules like cms, crm, epr etc. These modules will extend the functionality.

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
- VaahApp's top menu, left menu, profile dropdown on top menu and various other ui component should be extendable from other modules
- Frontend should support themes which can be and should be configurable from backend settings page
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
- **Portability**: a module built once runs on any VaahApp deployment (self-hosted by different businesses) without code changes, by targeting stable core extension points (see Extension Points).

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

## Modules
- New modules can be installed, enabled, disabled, upgraded, downgraded, deleted, with options to run migrations, seeds, etc.
- Each module can have its own MVC structure with migrations, seeds, notifications, emails, etc.
- Seeds are divided into two: 1. Required — when a module is activated, these seeds must run; 2. Sample — these seeds contain sample demo data.
- When a module is activated, it runs the migration and required seeds.
- **Extending workflows and agents**: a module's seeder can register workflows and agents into the platform (each stored as a record tagged with its owning module); they follow the module lifecycle — created/enabled on activation, updated on upgrade, and disabled/removed on deactivation or deletion.
- **Detach on edit**: once a user edits a module-seeded workflow or agent, it detaches and becomes user-owned — it no longer follows the module's lifecycle and survives the module's downgrade or deletion. A clear "owned by <module>" indicator shows the origin.
- **Distribution (later stage)**: a module registry / marketplace for distributing modules — publishing, discovering, and installing (distinct from the frontend module registry used for UI extension) — hosted at vaah.ai, planned for a later stage; until then, modules are installed directly (e.g. from a package/URL).

## Licensing & Distribution
- **Core is open source**: the VaahApp core platform ships open source; commercial value lives in modules.
- **Independent module licensing**: each module carries its own license — open source, freemium, premium, or SaaS.
- **Mixed-licensing runtime**: the core installs, enables, upgrades, and runs modules of different license types side by side; license type is a per-module property.
- **Entitlement enforcement**: the module lifecycle supports license checks — premium modules verify a license/entitlement, freemium modules enforce feature or seat caps, SaaS modules authenticate to a remote service. The core provides the enforcement hooks; billing/entitlement logic stays with the module/vendor.
- **No core metering/billing**: the core does not meter or bill (BYO LLM key, see Settings).
- **Isolation**: a module's code and proprietary/licensed logic stay contained within the module (HMVC); the core exposes stable extension points, not module internals.

## Extension Points / Module Extension Model
The core exposes a stable, documented set of extension points so a module (built by a third-party developer) plugs in cleanly. This is "extremely high extendability" made concrete:
- **UI shell**: contribute top-menu items, left-menu items, the profile dropdown, routes/pages, and dashboard widgets (via the frontend module registry).
- **Data**: add entities via migrations; required + sample seeds; use the shared storage layer for files.
- **RBAC**: register permissions and roles for the module's entities/actions (see RBAC).
- **API + MCP**: contribute REST/GraphQL endpoints and MCP tools, reusing the shared service layer so RBAC, audit, and attribution apply automatically.
- **Workflows**: contribute triggers, actions, and step types to the automation engine (see Workflows).
- **Agents**: contribute named agents with scoped tools and permissions (see Agents).
- **Settings**: contribute entries to the admin Settings page.
- **Notifications / email**: register notification types and email templates via the platform's notification system.
- **Events**: emit/subscribe on the event bus that drives workflow triggers and real-time updates.
- **Core capabilities**: consume — and where designed, extend — media, taxonomy, audit, logging, and reports.
- **Lifecycle**: install/enable/disable/upgrade/downgrade/delete, following the module lifecycle (see Modules).
- **Stability**: these extension points are stable across versions (semver); modules never reach into core internals (see Licensing → Isolation).

## CRUD Scaffolding
Whenever a CRUD is scaffolded, it automatically includes:
- **AI-assisted scaffolding**: a new module or CRUD can be scaffolded from a natural-language description (vibe coding) via the spec-driven build-time workflow (see Agent-Native Platform).
- **RBAC**: permissions and roles generated for that entity.
- **Migration** with standard columns: `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by`, `is_agent`, `agent_identity`, and an active/inactive status.
- **Agent attribution**: when an agent performs a change on behalf of a user, the record stores both the accountable principal (`created_by`/`updated_by`/`deleted_by` = `user_id`) and the instrument (`is_agent` + `agent_identity`, plus a link to the agent session/trace). `is_agent` reflects the most recent mutation. Applies to create, update, and delete.
- **Agent attribution in UI**: agent-made changes are identifiable on the dashboard — a row-level badge, a filter for agent-made changes, and attribution in the item/audit detail ("by Agent X on behalf of User Y").
- **API endpoints**: CRUD endpoints (REST + GraphQL) generated for the entity.
- **MCP**: the entity's operations are exposed as tools on the VaahApp MCP server.
- **Soft delete as default** (via `deleted_at`), not hard delete.
- **UI actions**: restore a deleted record, and permanently delete it.
- **Form actions**: save, save and new, save and clone.
- **Filters**: include trashed, only trashed, only inactive, only active.
- **Sorting**: multiple sort options.
- **Views**: efficient UX for list view, item view, and a quick item view (modal).
- **Trashed items** highlighted in light red.
- **Search**: debounced search plus full-text search.
- **AI form fill**: option to auto-fill the create or edit form.
- **AI search**: AI-powered search.
- **Real-time**: an edit by one user reflects immediately to all users.

## Audit Logs
- Global audit logs for every record and settings change.
- Audit logging is a core capability that modules can use or extend.
- A dedicated page to view all logs, with various filters.
- **Agent attribution & bulk operations**: audit entries record dual attribution — the user principal, the agent instrument, and a link to the agent trace. A bulk agent action produces one operation-level entry (e.g. "Agent X, on behalf of User Y, disabled 214 users in country Z") linking to the affected records, in addition to the per-record entries.

## Settings
A settings page (admin) to configure:
- **AI features**: enable or disable.
- **LLM details**: configure the LLM provider, model, and credentials used by the agent.
- **Date & time format**: a global format used across the platform.
- **Custom user profile fields**: add custom fields to extend the user profile.
- **2FA**: option to enable two-factor authentication.
- **Social login**: configure providers (Google, Facebook, LinkedIn).
- **Upgrade**: option to upgrade VaahApp (core).

## Registrations
- A `registrations` CRUD.
- Frontend registration/signup stores data in `registrations`, not directly in `users`.
- On successful verification, the record moves from `registrations` to `users`.
- Purpose: keep unverified/dummy data out of the `users` table.

## Media & Taxonomy
- **Media manager**: a core capability for managing media (upload, organize, reference) with on-demand transformations (resize/reformat) and CDN-ready delivery — usable in VaahApp and its modules.
- **Nested taxonomy manager**: a core capability for managing nested/hierarchical taxonomies (e.g. categories with subcategories) — usable in VaahApp and its modules.

## Data & Storage
- **File/media storage backends**: the platform stores files on one of two backends — **local filesystem** (files on the server) or **S3-compatible object storage** (AWS S3, MinIO, DigitalOcean Spaces, Cloudflare R2, …).
- **Storage abstraction**: the media manager and all modules read/write files through a storage layer that is backend-agnostic, so the backend is a configuration choice, not a code change.
- **Chosen at onboarding**: the storage backend (local vs S3) is selected in the onboarding/setup wizard; for S3, the endpoint, credentials, and bucket are captured at that point. The choice is stored as platform configuration.

## Notifications & Email
The platform's notification and email system — used by the core, modules, workflows, and system alerts.
- **Notification model**: a notification is a typed message targeted at a user or a set of users (e.g. a role or tenant), with a payload and the channels to deliver it on.
- **Channels**: in-app (admin dashboard), email, and push notification; SMS/webhook as optional additions. A notification can fan out to multiple channels.
- **Email sending**: transactional (single, immediate) and bulk (queued) email; delivered via **SMTP** (self-hosted) or an API provider (AWS SES, SendGrid, Postmark). SMTP/provider credentials are held by the backend (see the security boundary — agents/workflows never hold SMTP credentials).
- **Templates**: notifications and emails are driven by templates edited in a visual editor, supporting variables (e.g. user name, record values) and versioning; the email/notification customization from Features lives here.
- **Preferences**: per-user channel opt-in/out (and, where relevant, quiet hours).
- **Reliability**: delivery is async via the queue (Redis), with retries, delivery/bounce/complaint status, and rate limiting to prevent spam/abuse.
- **Extension**: modules register their own notification types and email templates (see Extension Points); system alerts (e.g. job failure, health degradation) reuse this system (see Observability).

## Reports
- Configurable reporting/BI sections (like Metabase): build dashboards, charts, and reports over platform data via configuration, not code.
- Reports can be shared, scheduled, and embedded in dashboards or modules.

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
  - VaahApp MCP server: CRUD and domain operations (generic tools + schema resources + journey prompts), reusing the same backend services as the REST API.
  - GenicUI MCP server: UI rendering (render / update / subscribe), provided by GenicUI and used as-is (not re-implemented in the VaahApp server).
- **Shared identity across both servers**: the domain and UI servers act under one shared identity — the same user token, RBAC, audit, and `is_agent`/`agent_identity` attribution. The UI server never runs with a different or elevated identity than the domain server; the agent is the intermediary (the servers do not call each other).
- Component model (chosen per capability):
  - Capability components: self-contained, own business logic, semantic props (e.g. UserSearch accepts a query prop and fetches its own data).
  - Presentational components: agent-fed data (e.g. UserDetail receives user data from the agent).
  - Components are authored in React (reused from the dashboard) and wrapped in a thin Web Component (a `GenicElement` subclass that renders the React component into a closed Shadow DOM) for GenicUI; the same React components are packaged as MCP Apps for cross-host rendering. One React component library, two packaging targets.
- Component catalog: every registered component carries a when-to-use description, a prop schema, and an example, so the agent can discover and route to the right component.
- **UI rendering surfaces (dual)**:
  - **In-app**: GenicUI renders components inside the VaahApp dashboard chat (deep AG-UI integration, shared state, theming, real-time).
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
VaahApp is agent-native: it is designed so that AI agents — the built-in runtime agent, external coding agents, and other systems' agents — can understand and operate it with minimal hand-holding, each via an open standard.

### Three agent surfaces
- **Runtime** — operate the live app: the built-in chat agent performs CRUD/domain operations and renders UI (see Agentic Platform, Agent Operations).
- **Build-time** — develop on the platform: coding agents (e.g. Claude Code, Copilot) scaffold and extend modules, CRUDs, and custom code on VaahApp.
- **Inter-agent** — agents delegating to/from VaahApp: external agents treat a VaahApp agent as a peer.

### Full MCP surface (VaahApp MCP server)
Beyond tools, the server exposes the full MCP capability set:
- **Tools**: CRUD/domain operations, reusing the same backend services as the REST API.
- **Resources**: the data model, schemas, and documentation exposed for progressive disclosure. Per-entity schemas use resource templates (parameterized URIs); annotations (`audience`, `priority`) steer the model.
- **Prompts**: reusable multi-step "journeys" exposed as slash commands.
- **`instructions`**: server-level standing guidance returned at initialize, describing how to use the whole server (tool specs as prompts).
- **MCP Apps**: VaahApp components can render as sandboxed-iframe HTML apps inside any MCP host (Claude Desktop, VS Code Copilot, M365 Copilot, …) via a `ui://` resource, with bidirectional tool/data flow.

### A2A (agent-to-agent)
- VaahApp exposes itself as an A2A agent with an **Agent Card** (a JSON manifest of capabilities, skills, and connection info served at a well-known URL) so external agents can discover it and delegate tasks.
- Supports the A2A task lifecycle, including an **`input-required`** state for native human-in-the-loop.
- Transports: JSON-RPC 2.0 over HTTP(S), with SSE streaming and async push notifications.
- Complements MCP: MCP lets an LLM use VaahApp's tools; A2A lets another agent delegate a whole task to a VaahApp agent as a peer.

### Spec-driven build-time agents
- "Vibe coding" a module/CRUD follows a spec-driven workflow: **constitution → specify → plan → tasks → implement → converge**, each artifact reviewed before the next, with a verify loop until converged.
- A **CLI** and agent-readable conventions (an `AGENTS.md` / `VAAGENTS.md`) let coding agents scaffold and extend VaahApp projects the way they work on any modern repo.
- Build-time agent work is auditable and repeatable — spec/plan/tasks/convergence reports are persisted — not a freeform prompt.

### Self-description layer
- **`llms.txt`** for the docs/website: a compact, agent-oriented overview plus links to Markdown page twins.
- **OpenAPI 3.1 + JSON Schema** for the REST/GraphQL API: machine-readable endpoint and data discovery (see API Design Conventions).
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
- **Providers & BYO keys**: supports multiple LLM providers; keys are BYO, stored encrypted (see Security & Data Protection), and can be set per tenant (see Multi-tenancy).
- **Model routing & fallback**: select a model per task/feature (e.g. a fast model for extraction, a capable model for reasoning), with automatic fallback to a secondary provider/model on error, timeout, or rate-limit.
- **Prompt management**: system prompts and tool instructions are versioned, editable, and testable; changes are auditable and reversible.
- **Guardrails**:
  - **Input**: prompt-injection defense (data returned by operations is treated as data, never as instructions — see Agentic Platform); PII redaction before content is sent to a model where configured.
  - **Output**: model output is validated against the expected schema/contract before it drives an action; malformed or unsafe output is blocked or sanitized.
  - **Actions**: agents can only act through the shared service layer (RBAC, audit, attribution) — a model response can never grant itself privileges (see Agentic Platform, RBAC).
- **Telemetry**: per-operation token/latency/cost telemetry feeds Observability; the core does no metering or billing (see Licensing & Distribution).
- **Model-agnostic**: no lock-in to a single provider; the model layer is swappable without changing the rest of the platform.

## Platform
- **Multi-tenancy**: see the dedicated Multi-tenancy section.
- **Localization (i18n)**: see the dedicated Localization section.
- **Accessibility**: WCAG-conformant, keyboard-navigable UI.
- **Data retention & GDPR**: retention policies and right-to-be-forgotten / data deletion; scheduled backups, restore, and HA live in the Disaster Recovery section.
- **Webhooks / outbound integrations**: emit events to external systems.

## Multi-tenancy
VaahApp is **multi-tenant-capable**: one deployment can host one or many tenants (a tenant = a business/organization boundary). A self-hosted instance for a single business is simply the one-tenant case — no special code required.
- **Default**: a fresh self-hosted instance provisions a single tenant (that business); multi-tenant hosting is enabled by adding more tenants.
- **Data isolation**: **separate database per tenant** — each tenant's data lives in its own database (or schema), the strongest isolation, and makes per-tenant backup and GDPR deletion straightforward.
- **Tenant admin vs platform admin**: in multi-tenant mode, a platform-level admin manages tenants (provisioning, configuration), distinct from each tenant's own admins.
- **Per-tenant configuration**: branding/theme, domain/subdomain, settings, and an independent LLM provider + key (BYO) per tenant.
- **Namespacing**: files/storage, cache keys, and queues are namespaced by tenant.
- **Provisioning & deletion**: create a tenant (provision its database + config + admin); delete a tenant (isolate/drop its data per retention/GDPR, see Platform).
- **Strict isolation**: a tenant can never read or affect another tenant's data; the boundary is enforced at the data layer and in every query.
- **RBAC within tenant**: roles/permissions/users are scoped to a tenant (see RBAC, Users).

## Localization (i18n)
The core provides a **string-translation mechanism for UI strings**; modules reuse the same mechanism for their own UI strings. **Content translation is each module's responsibility** — the core does not translate user or module content.
- **UI string translation**: platform UI strings are externalized and translatable via a dedicated mechanism (translation files / i18n service, with a default language and fallback).
- **Module strings**: modules use the same mechanism to translate their own UI strings, so a module's interface localizes the same way the core's does.
- **Content translation**: user-generated and module content is translated (or not) by the owning module; the core provides no content-translation service.
- **Language selection**: selectable per user, with a per-tenant / deployment default.
- **Formatting**: locale-aware date/number/currency/timezone formatting, tied to the global Date & time format (see Settings).
- **RTL**: right-to-left layout support for RTL languages.

## API Design Conventions
The public API follows current, industry-standard conventions. **REST is the canonical, versioned contract; GraphQL is a read-optimized surface; both are described by OpenAPI 3.1.** The same conventions apply to module-generated APIs, so every entity's API behaves predictably.
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
- **GraphQL**: a read-optimized surface for the dashboard and agent (flexible queries, no over/under-fetching); shares the same service layer, RBAC, audit, and attribution as REST.
- **OpenAPI 3.1**: the single source of truth — auto-generated from the NestJS service layer; drives client codegen, docs, MCP resources, and agent tool discovery (see Agent-Native Platform).
- **Auth**: OAuth 2.1 + OpenID Connect — client-credentials for applications, authorization-code + PKCE for users (see API Authentication).

### Agent-friendliness
The API is designed so that AI agents can discover, call, and recover from it with minimal hand-holding — the same machine-readable surface that the MCP tools are built on.
- **Self-describing**: OpenAPI 3.1 + JSON Schema (and the MCP resources) describe every endpoint, parameter, and response shape, so an agent can discover "what can I do here?" from the platform itself — no per-endpoint hand-holding.
- **Predictable & uniform**: consistent URIs, resource shape, filtering, and pagination mean an agent that learns one endpoint generalizes to the rest (fewer hallucinated paths/params).
- **Machine-actionable errors**: RFC 9457 `type` + field-level validation errors let an agent branch programmatically — re-auth on 401, pace/back-off on 429, correct and retry on 422 — instead of parsing free-text.
- **Safe retries**: idempotency keys let an agent retry a create/update without duplicates; cursor pagination is stable across retries (no offset drift).
- **Efficient**: `ETag`/304 and `Cache-Control` let an agent avoid re-fetching unchanged data (token/latency savings); rate-limit headers let it pace itself.
- **Consistent across surfaces**: REST, GraphQL, and MCP tools all reuse the same service layer, so behavior, RBAC, audit, and attribution are identical regardless of which surface an agent uses.
- **Stable**: versioned, backward-compatible contracts mean agents and integrations don't break on minor changes.

## API Authentication
API authentication is built on **OAuth 2.1 + OpenID Connect** (see API Design Conventions), with these access modes:
- **Application API**: application-level access for external applications/integrations — OAuth 2.1 **client-credentials** (or a long-lived API key as a simpler option), scoped by the application's RBAC permissions.
- **User tokens**: user-level access, two modes — **OAuth 2.1 authorization-code + PKCE** (delegated, short-lived access tokens) and **personal access tokens** (long-lived, scoped); both scoped to that user's RBAC permissions.
- **Enterprise SSO / IdP**: SAML and OIDC federation (e.g. Okta, Azure AD, Google Identity) for enterprise sign-in.

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
  - Dependency & SCA (software composition analysis) vulnerability scanning in CI; known-vulnerable dependencies are flagged.
  - Safe defaults: least-privilege, fail-closed access, and no debug endpoints in production.
- **Session & token management**: session timeouts, concurrent-session control, and token revocation (see API Authentication).
- **Module responsibility**: modules inherit the platform's security model (RBAC, encryption, secrets, audit) and cannot bypass it.

## Observability
Platform-level observability (distinct from agent observability in Agent Operations) for debugging and running the platform in production.
- **Logging**:
  - Structured application + error logs with levels (debug/info/warn/error) and a configurable log level.
  - A logs view in the dashboard to inspect and filter error and application logs (by level, time, request, user/tenant, module).
  - Correlation: every request carries a correlation/trace ID so a request can be followed across the service layer; agent and workflow actions reuse the same trace (links to agent/workflow traces).
  - Error capture: unhandled exceptions with stack traces and context, surfaced for debugging.
  - Modules emit logs through the same facility.
- **Monitoring**:
  - A status/monitoring view: overall system health, per-component health, and scheduled jobs (cron/workflows) — next run, last run, success/failure, duration.
  - Dependency monitoring: continuous checks on infrastructure — DB connectivity, cache (e.g. Redis), message queue, and external services — with status history and alerts on degradation.
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
- **Real-time**: efficient WebSocket/SSE fan-out — only the relevant updates reach the relevant users/tenants, not a broadcast to everyone.
- **Scalability**: a stateless API tier that scales horizontally, DB connection pooling, and queue-based background processing for heavy work (monitored via Observability).
- **Metrics**: performance metrics (latency, error rate, saturation) surfaced in Monitoring.

## Deployment & Self-hosting
Self-hosting is the primary, first-class deployment model: an organization runs VaahApp on its own infrastructure (VPS, VM, private cloud, on-prem, or container orchestrator). A managed offering may exist later, but self-host is the priority and the one that must always work.
- **Packaging**: containerized (Docker / Docker Compose) as the default install path, plus bare-metal/VM (Node.js) and Kubernetes for scale; a documented, reproducible install.
- **Configuration**: configured via environment variables / config (app URL, DB connection, storage backend, LLM provider + key, secret, …); the onboarding wizard handles first-run setup, ongoing config is env-based.
- **Dependencies**: Node.js (NestJS), a relational database (e.g. PostgreSQL), Redis (cache + queue), and a storage backend (local or S3, see Data & Storage); an LLM provider (BYO key). Minimum requirements documented.
- **Self-contained core**: the core is fully functional self-hosted without phoning home to a central VaahApp service. Remote calls are limited to optional items (update/upgrade checks, premium/SaaS module license validation).
- **Upgrades**: self-service upgrade path — pull the new version, run migrations/seeds, restart; documented and reversible (see Settings → Upgrade).
- **Data sovereignty**: all data (DB, files) stays on the user's infrastructure; exports and backups are available (see Platform).

## Configuration & Feature Flags
Centralized, environment-aware configuration plus feature flags for controlled rollout.
- **Configuration**:
  - Deployment-time config via environment variables / config (see Deployment & Self-hosting), validated against a typed config schema.
  - A runtime config surface in the admin console for settings that change without redeploy (see Settings), with per-tenant overrides where applicable.
- **Feature flags**:
  - Enable/disable features (core or module) without redeploy — scoped per deployment, per tenant, or per user/role.
  - Gradual / canary rollout: roll a feature out to a subset first, then broadly.
  - Flags are visible (which are on, where) and can be toggled from the admin console or API.
- **Module gating**: flags can gate a module's features or its activation, complementing Licensing & Distribution entitlements.
- **Safe rollout**: a flag change is reversible; where destructive it requires confirmation; changes are audit-logged.

## Disaster Recovery
The platform must survive failures and be recoverable, with defined recovery objectives.
- **High availability**: a stateless app tier that scales horizontally (see Performance); database and cache offer HA options (replication / failover) appropriate to the deployment.
- **Backups**: scheduled full + incremental backups of databases and storage (see Platform), off-site / cross-region where required.
- **Recovery objectives**: defined RPO (maximum data loss) and RTO (maximum recovery time) targets, set per deployment.
- **Restore**: tested, documented restore procedures (point-in-time where supported) — a first-class operation, not an ad-hoc script.
- **Failover & graceful degradation**: clear failover behavior for DB, cache, and queues; the platform degrades gracefully (queues buffer, cache misses fall through to the DB) rather than failing hard.
- **Migration safety**: upgrades and schema migrations are guarded (backup + dry-run) and reversible where possible to prevent data loss (ties to Deployment & Self-hosting upgrades).
- **DR testing**: periodic restore / failover drills to validate RPO/RTO.

## Tech Stack
- NestJs as Backend and API
- React + Vite SPA as Frontend
- Shadcn/ui as frontend component library
- TanStack Query (server/real-time) - Zustand (global client + the module registry).

## Terminologies
A glossary of the key terms used throughout this document, grouped by area. It covers both the **industry-standard terms** the platform is built on (e.g. MCP, A2A, RBAC, OAuth 2.1, cursor pagination, RPO/RTO) and **VaahApp-specific terms** (e.g. dual attribution, detach on edit, the frontend module registry) — the latter are defined here so all readers share one vocabulary.

### Architecture & platform
- **HMVC (Hierarchical Model-View-Controller)**: a modular architecture where the core and each module have their own MVC (model-view-controller) structure.
- **Modular monolith**: a single deployable application composed of strongly-bounded internal modules — the overall style of VaahApp.
- **Core**: the open-source platform foundation that modules install into and extend.
- **Tenant**: a business/organization boundary; in multi-tenant mode, the unit of data isolation (see Multi-tenancy).
- **Service layer**: the shared NestJS layer every caller (UI, API, agent, workflow) goes through; it enforces RBAC, audit, and attribution (see Agentic Platform → Security boundary).
- **Extension point**: a stable, documented surface where a module plugs into the core without touching internals (see Extension Points).
- **Self-hosting**: running VaahApp on your own infrastructure — the primary deployment model (see Deployment & Self-hosting).

### Modules & extension
- **Module**: an installable unit (e.g. cms, crm, epr) that extends the core; it ships its own MVC, migrations, seeds, UI, permissions, and MCP tools.
- **Frontend module registry**: the UI-side registry modules use to contribute top/left menu items, routes, and dashboard widgets.
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
- **MCP (Model Context Protocol)**: the open standard for exposing tools, resources, and prompts to LLMs; VaahApp exposes an MCP server.
- **MCP Apps**: an MCP extension — components rendered as sandboxed-iframe HTML apps inside any MCP host (via `ui://` resources in a sandboxed iframe).
- **A2A (agent-to-agent)**: the protocol for one agent delegating a whole task to another; VaahApp exposes an **Agent Card** (a JSON manifest of capabilities and connection info).
- **BYO LLM**: bring-your-own LLM provider + key (no core metering/billing).
- **Guardrails**: input/output/action safeguards on the model layer — prompt-injection defense, PII redaction, output validation (see Model Management & Guardrails).
- **Dual attribution**: recording both the accountable user principal (`created_by`/`updated_by`/`deleted_by`) and the agent instrument (`is_agent` + `agent_identity`) on every agent-made change.

### Data & CRUD
- **CRUD scaffolding**: auto-generating an entity's endpoints, UI, RBAC, migration, and MCP tools in one step.
- **Soft delete**: deletion via `deleted_at` (recoverable), not a hard delete.
- **Migration**: a versioned schema change applied when a module is activated or upgraded.

### API
- **REST**: the canonical, versioned API contract (`/v1/...`).
- **GraphQL**: a read-optimized API surface for the dashboard and agent.
- **OpenAPI 3.1**: the machine-readable API description — the single source of truth for codegen, docs, and MCP.
- **RFC 9457 Problem Details**: the standardized HTTP error format (`application/problem+json`).
- **Cursor pagination**: stable, token-based list pagination (`page_token` / `next_page_token`).
- **JSON:API**: the resource-shape conventions (`data`/`links`/`included`, resource objects) adopted for a uniform API envelope.

### Security & auth
- **RBAC (role-based access control)**: access control via roles → permissions → users; verified at the code level before every operation.
- **Field-level permissions**: read/write control at the column/field level, not just the entity level.
- **OAuth 2.1 / OIDC**: the authentication standard — client-credentials for applications, authorization-code + PKCE for users.
- **PAT (personal access token)**: a long-lived, scoped user token.
- **Idempotency key**: a token that lets a create/update be retried safely without duplicates.
- **Security boundary**: agents and workflows never hold infrastructure credentials; all actions run through the shared service layer.

### Workflows & automation
- **Workflow**: a deterministic, fixed-path automation (trigger → steps → actions) with no LLM in the execution loop.
- **Trigger**: what starts a workflow — event, schedule, manual, or API/MCP.
- **Run-as identity**: the identity a workflow runs under (event-driven defaults to the triggering user; scheduled uses a system identity).
- **Visual builder**: the Zapier-style, top-to-bottom step editor for building workflows.

### Operations & deployment
- **Feature flag**: a toggle to enable/disable a feature without redeploy, scoped per deployment/tenant/user.
- **Observability**: platform logging, monitoring, and health (distinct from agent observability).
- **RPO / RTO**: recovery point objective (max data loss) and recovery time objective (max recovery time).
- **Multi-tenancy**: multi-tenant-capable, single-tenant by default, with a separate database per tenant.
- **i18n / localization**: the core's UI string-translation mechanism, reused by modules.
