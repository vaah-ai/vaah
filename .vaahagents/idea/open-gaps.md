# Vaah — Stack Decisions & Open Gaps

*Companion to [overview.md](overview.md). This file records **decided** stack choices and the
**open** gaps still to be discussed one by one.*

- **DECIDED** — a call has been made; needs to be reflected in `overview.md`.
- **OPEN** — no call yet; discussed in order, one per session.

Every version below was verified against the npm registry or the package source on 2026-10-11.
Pinned versions live in `STACK.md` + lockfiles, **not** in `overview.md` — a requirements doc that
carries version numbers goes stale.

---

## Part 1 — DECIDED

### D-1 · OpenAPI 3.0.x via `@nestjs/swagger`
**Decision:** the platform's machine-readable contract is **OpenAPI 3.0.x**, generated code-first
from the NestJS service layer.

**Verified:** `@nestjs/swagger@12.0.2`, `@nestjs/core@12.1.2`. The package hardcodes a **3.0.0
default** — `dist/swagger-module.js:44` reads `config.openapi ?? '3.0.0'`. A `3.0.0` literal is the
only version string in the bundle. An override *is* possible (`SwaggerModuleOptions.openapi`,
`DocumentBuilder.setOpenAPIVersion()`), but 3.1 emission is not the supported path and would be
unverified.

**Consequence for `overview.md`:**
- [L342](overview.md#L342) and [L277](overview.md#L277) say "OpenAPI **3.1**" — change to **3.0.x**.
- [L513](overview.md#L513) glossary entry — same change.
- [L451](overview.md#L451) standards-alignment list — same change.
- `JSON Schema` alignment claims must be qualified: 3.0.x `nullable`-style schemas are not
  identical to JSON Schema 2020-12. State what it actually buys us.

### D-2 · MCP SDK v2 — **REVISED against the 2026-07-28 spec**
**Decision:** build the Vaah MCP server on **`@modelcontextprotocol/server`** (v2 scoped packages).

**Verified 2026-10-12:** `@modelcontextprotocol/server@2.3.1` (published 2026-04-01, last modified
2026-10-05) is the current version; legacy `@modelcontextprotocol/sdk@1.32.1`. **No package upgrade
needed — v2 was already the right choice. What changed is the specification underneath it.**

**The 2026-07-28 MCP specification rewrite is the largest revision since launch.** Consequences for
Vaah:

1. **The protocol core is stateless.** No `initialize` handshake, no `Mcp-Session-Id`, no hidden
   session context carried across calls — **auth and context travel explicitly as headers on every
   request.** *This settles a question the requirements never had to ask: the Vaah MCP server holds
   no protocol session state. All state lives in Vaah's own database and in the identity it carries
   per request.*
2. **MCP servers are formally OAuth 2.1 resource servers**, tightened considerably in this release.
   This aligns rather than conflicts with Vaah's existing OAuth 2.1 decision (see API
   Authentication) — the MCP endpoint is an additional access mode on the same identity and RBAC
   system, not a parallel one.
3. **Roots, Sampling, and Logging are deprecated**; **extensions are first-class**. MCP Apps
   (`ui://` resources) now ride on the extensions framework rather than being core. The
   [Agentic Platform](overview.md) section should describe MCP Apps as an **extension** — the
   January framing ("the first official MCP extension") is accurate but predates the rewrite.
4. **Existing capabilities do not need recertifying**, so MCP Apps adoption is not reset by the
   rewrite.

**Unchanged, and still the hard requirement:** client support for `ui://` remains uneven, and
[security analysis of the new spec](https://www.backslash.security/blog/new-mcp-spec-opens-new-attack-surfaces)
flags MCP Apps as an attack surface. **The Vaah MCP server must be fully functional when a host
ignores the `ui://` resource** — any data the model needs must also be present in the tool result.
This is a design constraint on the Agentic Platform section, not an implementation detail.

v1's support window closes around January 2027 — no new work on the old package.

### D-3 · NestJS 12 — HMVC stays a must-have capability
**Decision:** NestJS 12 (ESM-first) is the backend, and **HMVC remains a hard requirement**. The
core does not become a flat monolith; modules keep their own MVC trees.

**Verified:** `@nestjs/core@12.1.2` is `latest`; v11 is on the `legacy` tag; a `6.0.0-rc.1` tag
also exists.

**Correction to an earlier premise of ours.** We had recorded v12 as ESM-only with CommonJS dropped.
That is wrong — several blogs assert it, but the official migration guide says migrating *your*
application to ESM is optional and not part of the v12 upgrade, and this was verified by booting a
CommonJS v12 application. What actually changed is that **Nest's own packages ship as ESM**; a
CommonJS app consumes them via Node's `require(esm)`, unflagged from Node 20.19 / 22.12.

**Why this matters at requirements level:** "ESM-only" was never a constraint the stack imposed on
us — it is a **Vaah choice**. That does not make it wrong, but it means it should be defended as a
decision rather than inherited as a fact. *Consequence: the ESM-only module contract below is
policy we own, and it is the one place we impose a real cost on third-party module authors.*
Record the Node floor explicitly in the deployment requirements.

### D-4 · Real-time transport — **DECIDED: SSE**

**Decision: SSE for all server→client push, ordinary REST POST for client→server.** A single
multiplexed stream per browser tab. Full rationale and required mitigations in Appendix A.

**Requirements:**
- **No container is added** — this was a deciding factor under D-7. A WebSocket service would be one
  more thing the self-hosting developer runs.
- **Cookies carry the session.** `EventSource` sends cookies, so an HttpOnly session cookie works
  with no new code and the token never appears in a URL, referrer, or access log. *WebSocket also
  cannot set an `Authorization` header, so it buys no security advantage in exchange.*
- **At-least/at-most-once delivery is safe here because consumers are idempotent.** TanStack Query's
  `invalidateQueries` is idempotent, so a dropped message degrades to one missed refetch that
  self-heals. *Consistent with the job-queue idempotency requirement in O-6.*
- **Mitigations are part of the requirement, not optional hardening:** a heartbeat to keep proxies
  from closing the stream; `text/event-stream` excluded from compression; correct reverse-proxy
  buffering and read-timeout settings (the default proxy timeout will sever long-lived streams); a
  monotonic sequence guard for per-deployment ordering; **authentication in a guard rather than in
  the handler** — the response headers are flushed before the handler runs, so a handler-level check
  is too late; and **one refcounted stream per tab**, not one per component.
- **Redis pub/sub is lossy by design.** It is acceptable here precisely because delivery is
  best-effort and consumers refetch — but this must be stated, not assumed, or someone will build on
  it as if it were guaranteed.

**Agent chat progress streams over the same transport** ([L205](overview.md#L205)) — one mechanism for
record changes, notifications, dashboards, and agent output.

### D-5 · Search — **DECIDED: PostgreSQL full-text search for v1**

**Decision: PostgreSQL full-text search for v1, behind a `SearchProvider` interface.** Typesense
remains the intended later option, as an *optional* container with a working fallback if triggers
fire. Full rationale in Appendix B.

**Requirements:**
- **Search goes through a `SearchProvider` abstraction from day one**, even though the v1
  implementation is Postgres full-text. This is the requirement that makes a later swap possible at
  all — a search feature that calls Postgres directly everywhere cannot be replaced without touching
  every module. *This is the whole point of deciding the abstraction now and the engine later.*
- **No new container for v1.** D-7 was decisive: a dedicated engine is operational burden the
  self-hosting developer absorbs.
- **No cross-deployment result mixing.** An external shared index with a filter is a security
  regression against the isolation the platform otherwise provides; keeping search in the
  application's own database makes that structurally impossible rather than a thing to remember.
- **CRUD scaffolding wires search automatically.** Search applies per CRUD entity, so a generated
  entity is searchable with no extra work — the scaffolding template must include it, or the
  "everything is searchable" promise is not actually delivered by the standard path.
- **Migration triggers are documented** (Appendix B): volume, breadth, typo tolerance beyond what
  fuzzy matching gives, complex faceted latency, and search-as-headline-feature. When one fires, the
  `SearchProvider` seam is what makes the swap tractable — *if and only if it was actually used in
  v1.*
- **The optional later container must have a working fallback.** If Typesense is present, it is used;
  if it is stopped or removed, the platform still searches via Postgres. Optional infrastructure that
  is load-bearing is worse than none.

### D-7 · Self-hosted is total — the developer runs everything
**Decision:** a developer using Vaah to build their software is responsible for self-hosting
**everything, including all containers**. Vaah provides no managed or hosted component; the entire
dependency stack runs on the customer's own infrastructure.

This reinforces [L408](.vaahagents/requirements/overview.md#L408) ("no phoning home to a central
Vaah service") and makes it a hard boundary rather than a preference.

**Consequences:**
- **Container count and operational burden are a first-class product cost**, not an implementation
  detail. Every additional required container is something a non-expert self-hoster must get
  right, back up, monitor, and upgrade. This is the tiebreaker for D-4 and D-5.
- Dependencies must all support **offline / air-gapped operation** where the environment demands it.
- The onboarding wizard and `docker-compose.yml` must be excellent — first-run success rate *is* the
  product experience.
- Extension points must stay pure-HTTP/HTTP-hook based, so a module can integrate without the core
  having to run infrastructure on its behalf.
- Any optional infrastructure (dedicated search engine, and similar) should be **optional with a
  working fallback**, so a small deployment is never forced to run it.

### D-6 · Theming configured from the admin panel Settings page
**Correction to [L26](overview.md#L26).** The current text says *"configurable from backend
settings page"*, which understates the promise. Corrected wording:

> Frontend themes are authored and managed from the **admin panel → Settings** page and applied
> without a redeploy; per-tenant and per-user overrides where applicable.

This is a user-facing workflow, not a deployment-time config knob — the Settings page is already a
declared extension point (see [L93](overview.md#L93)), so themes become the worked example of it.

---

### D-8 · **Single-tenant — Vaah is one tenant per deployment** 🔴 *supersedes Multi-tenancy (L308–318)*

**Decision:** Vaah is **single-tenant**. One deployment serves one tenant. The multi-tenancy section of
the overview is superseded and must be rewritten, not amended.

**What this buys — and it is a lot.** Nearly every hard problem the multi-tenancy design generated
disappears:

- **Module activation is one migration run, not N.** The partial-failure problem, the per-tenant
  convergence question, and version skew across tenants (O-5b) all evaporate. Activation becomes
  validate → migrate → activate, against one database. This is the single largest simplification in
  the session.
- **Provisioning disappears.** "Create a tenant" was the mechanism by which new tenants received
  every active module's migrations. There is no longer a provisioning path to get right.
- **No tenant-scoped RBAC.** Roles, permissions, and users are simply roles, permissions, and users.
- **No per-tenant configuration surface.** Branding, theme, settings, and the BYO LLM provider/key
  are instance-level settings, not tenant-level records with a resolution layer.
- **No namespacing obligation** for storage, cache keys, or queues.
- **No GDPR-per-tenant deletion problem.** Data deletion becomes data deletion — one subject, one
  request, not an isolation exercise.
- **Cheaper per-request work.** No tenant resolution on every request, no tenant context propagation,
  no cross-tenant guard on every query.

**What it costs, stated plainly:**

- **Vaah is no longer a SaaS-building platform. It is a self-hosted application framework.** A
  developer who wants to serve many customers runs many Vaah instances. That is a coherent model —
  and under D-7 (the developer self-hosts everything) it is arguably the *natural* one — but it is a
  different product than the overview describes.
- **Per-tenant backup and restore becomes per-instance backup and restore.** Simpler, but it means an
  operator serving many customers runs many backup sets.
- **Some multi-tenancy work may be deferred rather than deleted.** If a future need appears, the
  cheapest path back is the **tenant-as-a-row** model (a tenant_id column plus row-level scoping),
  not the DB-per-tenant model that was specified. That is a much smaller change and a much weaker
  isolation guarantee. *Record this so a future reader knows the door is open and roughly which door
  it is.*

**Sections of the overview that assumed multi-tenancy and must be revisited:** Multi-tenancy (obvious)
· RBAC — scoping language becomes unnecessary · Settings — "per-tenant" becomes "instance-level" ·
Audit Logs — tenant scoping · Localization — unaffected · Data & Storage — namespacing ·
Disaster Recovery — per-tenant restore · D-7 and O-5b — both simplify sharply.

**Consistency check for the overview's own claim:** the doc says "a self-hosted instance for a single
business is simply the one-tenant case — no special code required." Under single-tenant, that becomes
*the only case* — and the claim is trivially true rather than an architectural achievement. Worth
saying that plainly in the rewrite, so it reads as a decision rather than an accident.

## Part 2 — OPEN (discuss one by one, in this order)

### O-1 · HMVC under NestJS 12 ESM 🔴 *blocking D-3* — **RESOLVED**

**Decisions:**

0. **HMVC is built in Vaah — it is not inherited from the framework.** NestJS provides dependency
   injection, module composition, and routing. It provides no hierarchical MVC, no model layer, no
   view layer, and no notion of a module owning its own tree. Everything HMVC means in Vaah is a
   convention Vaah defines, and every part of it is enforced by Vaah's own tooling or not at all.
   *This reframes the whole item: we are not selecting a pattern the framework supports, we are
   building one on top of a framework that has no opinion about it.*
1. **HMVC = one application, per-module MVC trees.** The core is a single deployable; every module
   — first-party and third-party — contributes its own model/view/controller structure *within* it.
   Modules are not separately deployed applications composed at runtime.
2. **Third-party modules are ESM-only.** Stated in the module authoring guide, enforced by the
   scaffolding CLI.
3. **Module endpoints are first-class API surface.** Every endpoint a module contributes must appear
   in the platform's OpenAPI contract, and therefore in agent tool discovery. *This is the load-
   bearing requirement.* A module invisible to the contract is a module no agent can drive, which
   breaks the extensibility promise and the agent-native thesis at the same time.
4. **The manifest is the import mechanism; the registry (DB) is the operational source of truth.**
   On first install the manifest (`module.json`) is read and registered into the module registry.
   From that point on the platform reads module data — the declaration, activation state, and
   everything the four outputs are generated from — from the registry, **not** by re-scanning the
   filesystem at boot. A module update re-reads the manifest and re-syncs the registry row.
   *This makes the active set a pure function of the registry read at boot, keeps all four outputs
   generated from a single source, and means the manifest is never a live read source — so it cannot
   drift from what is running. The manifest's only job after install is to be re-read on update.*

**What "HMVC" has to mean, given the framework supplies none of it.** These are ours to define, and
until they are defined the requirement is not actionable:

- **The tree shape.** What a module's model / view / controller directories are, and what each layer
  is *for*. NestJS's "module" means a DI scope; Vaah's means a self-contained vertical slice. The
  naming collision should be acknowledged — using the same word for two different things is a
  standing source of confusion in the docs and in module authors' expectations.
- ~~**The hierarchy.**~~ **DECIDED: one level.** A module may contain sub-modules; there is no
  recursion. "H" denotes *composable units at one level*, not inheritance from a core base. *The
  nesting is organisational — grouping files and routes — and keeping it one level is what keeps
  ordering, activation, dependency resolution, and the compatibility check one level deep instead of
  multiplying combinatorially.*
- ~~**The view layer is the genuinely unresolved piece.**~~ **RESOLVED by
  [O-16.4](open-gaps.md#L1544) and the [O-3](open-gaps.md#L369) package decision.** A module's views
  are runtime-loadable frontend bundles it registers into the admin SPA; modules ship as
  self-contained packages with their own build and assets. Backend-only modules remain possible — they
  simply register no frontend bundle.
- ~~**Model layer ownership.**~~ **RESOLVED by [O-5b](open-gaps.md#L498)**: a module owns its own
  migrations and tables, with mandatory up-paths, a dry-run pass, and rollback.

**Consequence — the scaffolding CLI is load-bearing, not a convenience.** With no framework
enforcement, the CLI's generated structure *is* the conformance mechanism for third-party authors.
Whatever the module contract is, the default path must produce a conforming module with no extra
effort from the author. This is the same conclusion reached from three directions: contract
completeness below, load-time validation in O-4, and the unsupported-pattern risk in O-4.

**Requirement-level consequences to record in the doc:**

- **Module lifecycle is restart-scoped.** Enable, disable, upgrade, and uninstall take effect on
  application (re)start. The module lifecycle section must state this rather than imply hot-swapping.
  *Confirm this is acceptable — it is the honest consequence of the HMVC model.*
- **A module's schema fault must not break the platform contract.** One module's malformed or
  unsupported schema may degrade *that module's* spec coverage; it must never fail spec generation
  for the whole platform. Module isolation extends to the contract surface, not only to data.
  *This matters more under D-7: a self-hosted instance that loses its API contract because of one
  third-party module is unusable, and the developer self-hosting it owns the failure.*
- **Enable/disable state is data, not code.** Held as a status record so it is reversible, auditable,
  and survives restarts.
- **Modules declare their own dependencies**, so bootstrap order and integrity are checkable rather
  than assumed. *Validation semantics — DECIDED:* a dependency must be **installed and active**, not
  merely present; each dependency carries a **version range** (not just a name), checked against the
  running version of the dependency; **circular dependencies are refused** at activation; and the
  **transitive** set (dependencies of dependencies) is surfaced to the operator, not only the direct
  `requires` list. *Deactivation cascade — DECIDED:* when a module is deactivated, any active
  module that depends on it (directly or transitively) is **also deactivated at the next boot** —
  the active set is recomputed as a closure at startup: a module is active only if every one of
  its dependencies is active. This is not a boot failure; the dependent is simply excluded from
  the active set for that run. The deactivation action must tell the operator which dependents
  will be disabled, and re-activating the dependency restores them on the next boot.
- **The Vaah CLI generates modules with a default UI — DECIDED.** `vaah module:create` produces a
  module that arrives with working, presentable UI, which the developer may then **completely
  customize**. This settles the "what does extendability mean" question: not distinct identities, but
  a complete working surface out of the box that can be replaced wholesale. See O-2 — it resolves
  most of that item.
  - *Requirements:* the generated UI must be real code the developer owns and edits, **not** a
    locked-down or configuration-driven layer. If customization means "fill in these slots," the
    promise is weaker than "completely customized" implies.
  - *The corollary that matters:* because the UI is default-generated and then replaced, a module
    that never touches the UI contributes **nothing** to the design-token namespace. The collision
    risk in O-2 exists only for modules that customize partially.

**Module manifest** follows the [laravel-modules](https://github.com/nWidart/laravel-modules)
pattern, studied directly: a self-describing `module.json` per module, a `requires` list, and an
explicit ordering field. The manifest is the import mechanism (decision 4 above) — read on first
install and on update, and registered into the module registry, which holds the declaration and
activation state as the operational source of truth. Vaah's manifest extends that pattern with a
core-compatibility range, ESM entry points, and a separation between sample/seed content and
required content. *The exact field set is a design decision for the module specification, not
settled here.*

**One requirement is *not* satisfied automatically — and it is the one that matters most.**
Requirement 3 splits in two. **Routes** from a runtime-loaded module do reach the generated
contract automatically — verified, no caveats. **Request/response schemas do not.** Without
explicit per-property declaration, a module's endpoint appears in the contract with an empty
response and no body shape.

For an agent-native platform this is the whole ballgame: an agent discovering a tool with no
parameters and no result shape cannot call it usefully. So this converts from an infrastructure
question into a **contract obligation**:

> **A module's endpoints must appear in the contract complete — route, parameters, request body,
> and response shape — or the module is not conforming.**

**Requirements that follow:**
- The module contract **mandates** explicit declaration of parameters, request body, response
  types, and every DTO field. Not optional, not best-effort.
- **Every controller declares its own tag.** Untagged controllers are auto-named from the class
  name, so across multiple modules tags collide — and in the contract, colliding tags mean an agent
  cannot tell whose tool it is.
- **A module with incomplete contract coverage is detectable.** Since we cannot rely on the
  generator, the platform needs a completeness check that treats "endpoint present, schema absent" as
  a conformance failure. This is the same "validate the module contract at load time" requirement
  that O-4 and Topic 2 both need. **Three topics, one requirement — it should be stated once,
  centrally, as a module-contract-conformance gate.**
- The scaffolding CLI generates conforming modules by default, so the obligation costs a module
  author nothing if they use the standard path.

*Recorded as not-yet-verified, explicitly **not to be built on**:* whether modules may instead **ship pre-generated metadata** alongside the module
rather than declaring schemas by hand. Two open concerns make this a live question rather than a
settled one — the merging behaviour of that metadata with hand-written declarations is reported
inconsistent, and whether our agent-facing tooling can consume multiple separate contract documents
rather than one is unverified. *Do not build on this path until checked against the current version.*

### O-2 · Design tokens & runtime theming 🔴 — **RESOLVED by the CLI decision**

**Resolved.** The CLI generates modules with a default UI that developers may completely customize
(decided above). This answers what "extremely high extendability" means: **a complete working surface
out of the box, fully replaceable** — not a distinct visual identity per module.

**That reframes the collision problem, and mostly dissolves it.** The original concern was that core
tokens and module tokens both emit globally and collide silently. But:

- A module that leaves the generated UI alone uses **only core tokens** and contributes nothing to
  the namespace. No collision.
- A module that fully replaces its UI also contributes nothing — it either uses core tokens or brings
  its own entirely.
- **Collision is confined to partial customization** — a module that customizes some screens but
  leaves others consuming core tokens, and that defines its own tokens for the customized parts.

So the requirement is narrow and checkable rather than open-ended: *a module that defines its own
design tokens must namespace them under its own identifier.* Full replacement is unproblematic;
partial customization is the only case that needs a rule. This also means the token contract's
stability burden is far lighter than it first appeared — modules mostly *consume* core tokens rather
than depending on their own remaining valid indefinitely.

**Still required regardless:**
- **The token contract is versioned.** Modules consume core tokens, so a core upgrade renaming or
  removing a token breaks them. This makes O-14 (versioning policy) a dependency of this item.
- **Shared token package as a `peerDependency`**, not a `dependency` — otherwise the package manager
  installs two copies and the design system silently duplicates.
- **Runtime swap must use `inline`** — without it the variable indirection breaks and themes cannot
  swap at all. Plus `[data-theme]` + `.dark`, and a blocking `<head>` script reading a **cookie**
  (not just localStorage) to avoid a flash of the wrong theme on load.

**DECIDED — tenant themes are partial overrides.** Core defaults always apply; a tenant supplies
values only for the tokens they choose, and everything they leave unset keeps a working default.

**Why this is the right call at requirements level:** it makes theme safety a **structural
guarantee rather than a validation best-effort**. A tenant cannot produce an unreadable interface,
because the tokens they do not set keep their values. Full palette would have made safety depend on
catching every bad combination, which no validation can do exhaustively. Under D-7 — the developer
owns the instance and there is no escalation path — the guarantee is worth more than the extra
expressiveness.

**Requirements this generates:**
- **There is a required-token set** that must resolve to a usable value on every page, whether
  tenant-supplied or defaulted. This set is what "valid theme" means, and it belongs in the token
  contract.
- **The admin Settings page validates on save** — against the required set and a value range — so
  the developer gets a clear error rather than discovering a broken palette later.
- **Token values are stored as tenant records**, resolved at request time against core defaults.
  This connects theming to the token-resolution layer and to O-5 (ORM) and multi-tenancy.
- **Documented as a deliberate v1 limitation, not an oversight.** A tenant cannot currently express
  a full custom palette. State that plainly, and record what would have to change to lift it —
  a resolution layer already exists, so full palette is an extension rather than a rewrite.

**Status: Topic 2 / O-2 is closed.** The only remaining dependency is O-14 (versioning policy),
since modules consume core tokens and a core rename breaks them.

### O-3 · Repo topology & module version skew — **DECIDED**

**DECIDED — module priority/order:**
1. **Every module carries a priority** determining its position in the extension and bootstrap order.
   This mirrors the ordering field in the laravel-modules manifest, which is the pattern Vaah's
   module manifest already follows.
2. **Vaah assigns the priority when a module is installed.** The developer does not set it at install
   time and cannot be left with an unset value.
3. **The priority is customisable from the UI afterwards** — reorderable by the developer once the
   module is installed, without touching the module package or a config file.

**Requirements this generates:**
- **Priority is instance state, not module state.** It is held alongside the enable/disable status
  record, so reordering does not require editing the module itself and survives upgrades of that
  module. A priority declared *inside* the shipped manifest would be overwritten by any module
  update; instance state is not.
- **The developer can see and change the whole ordering at once** — the ordering is only meaningful
  if the full sequence is legible, not just a per-module number.
- **Priority interacts with declared dependencies, and dependencies must win.** If module B declares
  it requires A, no amount of priority ordering may place B before A. The priority mechanism orders
  *independent* modules; it must not be able to violate a declared dependency. *This needs to be
  stated as a hard rule, because an ordering UI that lets a developer create an invalid sequence is a
  footgun that surfaces at boot.*
- **Ties need defined behaviour.** Two modules at the same priority must have a deterministic
  secondary sort (name, or install order) — an undefined order is a class of bug that only appears
  in production.

**DECIDED — repo topology and module distribution:**

1. **Option C — Vaah core in its own repo; modules in their own repos, under their own authorship.**
   Not a monorepo (a monorepo cannot express per-module read access, which forces core and
   third-party code into one permission domain), and not a single shared repo for everything.
   *This matches the decision already taken on distribution: modules are installed artefacts, not
   directories inside Vaah.*
2. **Modules are self-contained packages** — installed as archives (zip) or via npm, each with its
   **own build output and its own assets**. A module is a complete, independently versioned unit that
   carries its frontend bundle ([O-16.4](open-gaps.md#L1325)), its migrations, its permissions, and
   its manifest. It is not a source checkout that Vaah compiles.
   **Consequences, and they are the load-bearing part of this decision:**
   - **This confirms runtime-loaded module UI** ([O-16.4](open-gaps.md#L1325)) — a module shipping a
     prebuilt asset bundle *cannot* be bundled into Vaah at build time. The two decisions agree.
   - **The host must supply shared dependencies** (React, design tokens, query client), and a module
     declaring a conflicting version of one of those is **rejected at install**. *This is the failure
     mode nobody builds and it must be checked, not assumed: an unsatisfied dependency range makes
     the installer pull a second copy, producing duplicated singletons and duplicated state — two
     React instances, two query clients, two copies of anything else the host owns.*
   - **Install is atomic and validated before activation**: manifest present, compatibility range
     satisfied, dependency constraints satisfied, migrations declared with up-paths
     ([O-5b](open-gaps.md#L498)). A module failing any of these is refused at install, not
     half-loaded at boot.
   - **Core APIs a module may depend on are declared explicitly** in its manifest, rather than
     discovered by reaching into core — the mechanism that makes "modules never touch core internals"
     ([L98](overview.md#L98)) enforceable rather than aspirational.

3. **Versioning: independent per module, with Vaah's major as the contract boundary** — as already
   decided in [O-14](open-gaps.md#L1109). Restated here because the package shape makes it concrete:
   a module's own version moves freely; its *declared compatibility range* is what Vaah enforces at
   install and load.

**DECIDED — who verifies compatibility, and what "verify" means:**

4. **Vaah runs sanity checks at activation. It does not run the module's tests.**
   When an operator clicks **Activate** on the module listing page, Vaah validates that the module is
   **correctly formed and correctly declared** — nothing more. The bar is *"is this a valid module
   that properly registers with Vaah"*, not *"does this module work correctly"*.

   **The sanity checks are:**
   - the manifest is present, parseable, and schema-valid;
   - `vaah_compatibility` is declared and **satisfied** by the running Vaah version (O-14, enforced
     fail-closed at load);
   - declared dependencies are present and enabled, and no declared dependency is violated;
   - declared core APIs (the manifest's explicit dependency list) all exist in this Vaah version;
   - no conflicting version of a **host-supplied** dependency (React, design tokens, query client) —
     the duplicated-singleton guard from decision 2;
   - migrations are declared and every one has an **up-path** ([O-5b](open-gaps.md#L498));
   - declared permissions and routes do not collide with core's or another module's;
   - the frontend bundle is present and loadable where the module declares UI (O-16.4).

   **Explicitly NOT Vaah's job:** running the module's test suite, judging whether its behaviour is
   correct, or verifying that it does what its documentation claims. *This is the same boundary as
   [O-7](open-gaps.md#L712) and [O-12](open-gaps.md#L1003): Vaah defines and checks the **contract**,
   and does not take responsibility for module behaviour. Running a module's tests would require Vaah
   to understand an arbitrary module's test harness, its fixtures, and its dependencies — which is
   both infeasible and exactly the responsibility Vaah has already declined.*
5. **Behavioural verification is the module author's responsibility, run in the module's own CI.**
   Vaah provides no central compatibility test farm ([O-3](open-gaps.md#L369) option (c) declined).
   *Accepted limitation, recorded: a module can satisfy every sanity check, be correctly formed, and
   still be broken. The compatibility range catches **structural** incompatibility (an API that no
   longer exists); it does not catch a module that depends on behaviour Vaah changed without breaking
   any API. This is a trust boundary, and it is the same one the design already accepts everywhere
   else.*
6. **Activation is fail-closed and dry-run first**, consistent with [O-5b](open-gaps.md#L498): the
   sanity checks and the migration dry-run both pass before anything is applied. A module failing a
   sanity check is **not activated** and the operator sees which check failed.

**Where the sanity checks run matters for one thing:** they run **at activation, on the operator's
instance** — not at publish time on someone else's machine. So a module can be well-formed enough to
install and still fail activation, and the operator is the one who finds out. That is the correct place
for these checks, because activation is the moment Vaah actually needs the module to be valid.

#### Remaining loose ends — DECIDED (closing out the open list)

1. **Module hierarchy: one level. DECIDED.** A module may contain sub-modules; there is no recursion
   beyond that. *This closes the "HMVC — how deep is the hierarchy" question. The nesting is
   organisational (grouping related files and routes), not a recursive architecture — which keeps
   ordering, activation, dependency resolution, and the compatibility check one level deep, and those
   are the mechanisms that would otherwise multiply combinatorially.*
2. **Seeds are idempotent and duplicate-checked. DECIDED.** *Distinct from migrations, which are
   versioned schema changes: seeds are data.* The requirement:
   - **A seed declares how it identifies its records** — a natural key or explicit conflict column —
     so re-running it updates rather than duplicates.
   - **Re-activation re-runs seeds safely.** The failure mode this prevents is a module deactivated
     and reactivated producing duplicate sample data, which is exactly the kind of defect that only
     surfaces in a demo.
   - **Sample seeds are separated from required seeds** (already in the manifest design from O-1), so
     a module can ship demonstration content that is never applied on activation.
   - **Sample-data cleanup on reset is owned by the reset operation** ([O-5b](open-gaps.md#L498)) —
     reset removes migrations and associated media/taxonomies, and sample seeds fall under the same
     cleanup so a reset leaves no orphaned sample content behind.
3. **LLM/agent timeout: default adopted, configurable per deployment. DECIDED.** The *mechanism*
   (short timeouts for ordinary requests, a separate longer ceiling for agent calls, O-11) was
   already settled; only the value was open. Default: **5 minutes for the agent-facing LLM ceiling**,
   short timeouts unchanged for ordinary requests.
   *5 minutes is a middle choice, not a derived one — long enough for a substantial agent run with
   several tool calls, short enough that a genuinely hung provider is abandoned before an operator
   concludes the platform has frozen. It is a **default, not a constraint**: the value is configurable
   per deployment, precisely because an on-premise deployment doing long batch-style agent runs needs a
   different ceiling from an interactive chat product. The split between the two tiers matters more
   than either number.*
4. **Visual regression testing: advisory, not blocking. DECIDED.** *Contrast with accessibility
   ([O-12](open-gaps.md#L1003)), which is a blocking gate.*
   The reasoning is asymmetry of consequence: an accessibility failure **excludes users** and is
   objectively checkable, so gating is right. A visual diff is frequently subjective — a font metric
   difference, an antialiasing change, a two-pixel reflow from an unrelated dependency — and a blocking
   gate on a subjective diff trains people to re-run until it passes rather than to fix anything.
   **Recorded so it is not mistaken for indifference:** visual regression remains **reported on every
   change** with a link to the diff, so a reviewer can see what moved. It is advice, not silence.
5. **Pre-generated module schema metadata — not decided, and explicitly not to be built on yet.**
   *Explained plainly here for the record, since the original note was written in implementation terms:*
   the question is whether a module author must **hand-write schema declarations** (entity names,
   fields, response shapes — the annotations that let Vaah generate OpenAPI docs and MCP tools), or
   whether a **tool can generate those declarations for them** into a separate file the module ships
   alongside its code.
   **Why it stays open rather than being decided either way:** the generated metadata has to be
   *merged* with whatever the module declares in code, and that merge behaviour has been **reported
   inconsistent** — it has not been verified against the current framework version. Separately, it is
   unverified whether our agent-facing tooling can consume **several separate contract documents**
   rather than one.
   *Both are verification gaps, not design gaps. Until they are checked against the running version,
   specifying this would be specifying something that may not work. The existing path — hand-written
   declarations, generated for the author by the scaffolding CLI — is unaffected and remains the
   supported route.* **This is the correct state for an unverified implementation detail: recorded,
   not decided, explicitly flagged not to build on.**

### O-4 · Module trust model & marketplace curation 🔴 — **DECIDED (mirrors the WordPress plugin model)**

**The model: trust on install, with Vaah responsible only for registration.**

The user supplied the precedent directly, and it is the right one: **WordPress**. Anyone can write a
plugin and anyone can install one; WordPress core's responsibility is to check that it is **configured
correctly as a plugin** — a valid header, a valid entry point, no fatal load error — and then gets out
of the way. Nobody asks WordPress whether a plugin is well-written, safe, or does what it claims.
That ecosystem is the largest of its kind in the world and it works.

**Decisions:**

1. **Vaah verifies registration, not trustworthiness.** Exactly the [O-3](open-gaps.md#L369) sanity
   checks: valid manifest, satisfied `vaah_compatibility`, dependencies present and enabled, no
   conflicting host-supplied dependency versions, migrations with up-paths, no permission or route
   collision, loadable frontend bundle. **Vaah does not sign modules, does not verify author identity,
   and does not block on any judgement about quality, safety, or fitness for purpose.**
2. **No signature verification in v1.** *Rejected options (b)/(c) from the discussion.* Signing only
   catches a **tampered** download — it proves nothing about whether the author's own code is
   dangerous, since a valid signature is exactly what a malicious author would produce. It also needs
   a public-key registry with no purpose until third-party modules actually ship. *Building
   verification infrastructure before there is anything to verify is speculative.*
3. **No sandboxing — explicitly out of scope, not merely deferred.** Modules run in-process with the
   same database connection and application privileges. Real isolation would mean reimplementing the
   shared service layer across a process boundary, which directly contradicts the design: modules
   *reuse* the shared service layer precisely so RBAC, audit, and attribution apply automatically
   ([L90](overview.md#L90)). Isolation and that reuse are mutually exclusive. *This is the honest
   trade — Vaah buys uniform governance for modules by giving up isolation from them.*
4. **Marketplace curation: inform, not enforce** — the same answer as [O-12](open-gaps.md#L1003).
   Listings publish what the author declares; Vaah does not review, rank on quality, or reject.

**The one requirement added — declared capabilities are surfaced before install.** This is not a
security control; it is **visibility**, and it is what makes trust-on-install an informed decision
rather than a blind one. Before an operator confirms an install, Vaah shows what the module declares
it wants:

- network access (outbound calls)
- storage access (which backend, read/write)
- background/scheduled job capability
- routes and admin menu entries it will register
- **agents and the scoped tools they will carry**
- the Vaah version range it claims

and records in the **audit log** that the operator approved those capabilities.

**Recorded limitation — the accepted cost of this model.** A module can satisfy every sanity check,
register correctly, and be malicious or simply broken. Vaah provides no protection against this, and
the operator assumes it by installing. *Stated plainly because WordPress demonstrates the cost
precisely: the plugin ecosystem is also the largest recurring attack surface in that platform. The
argument for the model is not that it is safe — it is that it maximises ecosystem growth, keeps Vaah
small, and puts responsibility where the operator can actually discharge it. Any self-hosted platform
already requires its operator to trust the packages they install.*

**Vaah-specific amplification, and why the capability list above matters more here than it would for a
WordPress plugin.** Vaah modules can contribute **named agents with scoped tools and permissions**
([L89](overview.md#L89)) — a capability WordPress has no equivalent of, because a plugin adding a
dashboard page is not the same as a plugin introducing an autonomous actor that can call tools. A
module that ships an agent is asking for the highest-value capability in the system.

**Therefore: contributing an agent is surfaced explicitly at install, separately from the rest of the
capability list, and cannot ride along silently with the rest of the install.** *This is the one place
the WordPress analogy does not transfer directly, and it earns the extra requirement: the blast radius
of a misbehaving agent is bounded by permissions, but nothing bounds a human's willingness to click
"yes" on a module listing that looks reasonable.*

*Research findings below are retained — they establish why sandboxing was ruled out rather than
simply deferred, and they document the containment mechanism ([O-4 containment decision](open-gaps.md#L406))
that stands alongside this trust decision.*

Modules are **third-party code running in your process**, and the doc currently treats them as data.

**Now established by research — there is no sandbox, and isolation is by convention:**
- A module shares the **core DI container**. Global registrations (guards, interceptors, pipes,
  filters) declared by a module apply **platform-wide, including to core routes**. This is
  cross-module privilege escalation, not a theoretical risk.
- Loose-resolution lookups reach into core internals; `@Global()` modules leak app-wide.
- Singletons are **shared and mutable** — a module can mutate core configuration state, and core
  observes the change.
- Same-named dependency tokens **collide silently**. A module that declares its own data-source class
  may receive it under a name the core also uses. No error surfaces.
- `process.env`, `globalThis`, and module top-level side effects are shared and **not rolled back**
  if the module later fails to initialise.

**Requirements this generates:**
1. **Module load-time contract validation** — reject a module that registers global enhancers,
   declares global modules, or introduces colliding dependency tokens. This is a platform
   responsibility, not a module-author convention.
2. **A module must not be able to reach core internals** ([L98](overview.md#L98)) must be *enforced* —
   architectural test or lint rule, not documentation.
3. **Failure containment — DECIDED: skip and disable.** A module that throws during startup does not
   prevent the platform from starting. The platform **boots without it, and the module is disabled**.
   - **Log entry** for the failure, with the module identity and the error.
   - **Flash message in the dashboard UI**, so the developer sees it without reading logs. Under
     D-7 this is the primary channel — they may have no easy log access.
   - *Requirement:* the disabled state must be **visible and persistent**, not transient — surfaced in
     admin health so it is not mistaken for a module that was never installed.
   - *Note the contrast, which is deliberate:* **activation** is fail-closed (nothing changes, roll
     everything back). **Startup failure** is fail-open (boot without it). Activation failure happens
     before the platform is committed; startup failure happens after. Different risk positions,
     different correct answers.
   - *Open:* does a startup-failed module's enable-state record get set to disabled automatically,
     or does it stay enabled-but-failed? The first is simpler and honest; the second preserves the
     developer's intent so a later restart retries it. **Recommend: record it as failed, not
     disabled — retry on next boot.** Needs a call.
4. **Signing + verification** of a module package before install (supply-chain entry point).
5. **Marketplace curation/review** process.
6. **Containment — DECIDED: isolation with visibility.** Not full process sandboxing (which would cost
   the native route registration HMVC depends on), and not passive acceptance. Vaah provides
   **declared-capability isolation, surfaced to the developer**.
   - **Modules declare the platform capabilities they use** — database, outbound HTTP, filesystem,
     queues, scheduling, background jobs — in their manifest.
   - **The platform grants only what is declared**, and can deny any of it. A module that reaches for
     something it did not declare is refused, not merely discouraged.
   - **Visibility is part of the feature.** The admin console shows, per module, exactly what it has
     been granted and what it uses — so isolation is auditable at a glance rather than trusted.
   - **This bounds the blast radius of everything except runaway execution.** A module can still
     infinite-loop or leak memory; nothing in-process prevents that. **Document that as an accepted
     limitation**, plainly, in the module documentation — Vaah does not protect you from a module
     that misbehaves. What it does is ensure a module can only reach what it declared, and that the
     declaration is visible.
   - *Why this framing:* it is the honest middle. It gives a defensible line ("a module declares what
     it needs and can be denied access it didn't declare") without claiming a sandbox that does not
     exist.

*Strategic risk worth recording explicitly — Vaah's module model rests on a pattern the framework
does not endorse.* There is no official NestJS plugin architecture, no plugin documentation, and no
test coverage or filed issue anywhere describing our exact scenario (runtime filesystem discovery
composed into the root module). On two separate issues the framework's lead maintainer declined to
treat it as a supported use case, and one open request to support dynamic route removal has been
open since 2019. Everything we rely on was verified by execution, but by us, on a scratch
installation — **not sanctioned by a maintainer.** *Requirements consequence:* the module contract
should not depend on undocumented behaviour in more places than strictly necessary, and the
scaffolding CLI should carry module metadata through **explicit authoring** rather than inferring
it, so conformance survives even if upstream behaviour changes. This is an argument for making the
module contract explicit and self-describing — the same conclusion the contract-completeness
requirement in O-1 reaches independently.

*Note on the interaction with Topic 2 (design tokens):* the same convention-not-enforcement problem
appears there. A module shipping an unprefixed design token silently overwrites core styling. Both
topics reduce to the same requirement — **the platform must validate the module contract at load
time rather than trusting module authors.** Worth stating once, centrally, instead of twice.

### O-5 · ORM choice — **DECIDED: MikroORM**

**Requirement: an Eloquent-equivalent** — active-record-style models, a unit of work, an identity
map, first-class relations. Not a query-builder DSL over raw SQL (Drizzle, Kysely), not a
schema-first client generator (Prisma).

- **MikroORM** — `@mikro-orm/core@7.2.5`, `@mikro-orm/nestjs@7.1.0`, `@mikro-orm/postgresql@7.2.5`.
  This is the closest established match to Eloquent in the NestJS ecosystem: active record entities,
  identity map, Unit of Work with automatic change tracking, cascade persist.
- TypeORM also offers an active-record mode and is maintained (`typeorm@1.1.2`,
  `@nestjs/typeorm@12.0.2`), but its reputation is weaker and the identity-map behaviour is not its
  strength. MikroORM's change tracking is the closer analogue to what Eloquent does.

**Note the interaction with the single-tenant decision (D-8).** The original deciding factor for this
item was *cheap per-tenant clients*, which is now moot — there is one database. The choice rests
entirely on the Eloquent-equivalence requirement, and MikroORM satisfies it. This also removes the
per-tenant migration-state tracking that DB-per-tenant would have required.

### O-5b · Module-owned migrations & activation hooks 🔴 *newly decided, see below*

**Decisions:**
1. **A module may be backend-only.** API surface with no frontend contribution is a valid, fully
   supported module. The manifest records which parts are present rather than requiring all of them.
2. **Each module owns its migrations.** A module defines its own schema changes and they are applied
   and tracked as that module's own — not merged into a platform-owned global migration history.
3. **Module lifecycle actions emit lifecycle events**, and the platform runs the module's migrations
   and seeds in response. Activation is the trigger point named so far; the full set of lifecycle
   events is open.

**This collides with DB-per-tenant, and that collision is the whole substance of this item.**
Vaah is DB-per-tenant ([Multi-tenancy](overview.md)). So "run the module's migrations" is not one
migration — it is *the same migration against every tenant's database*. The consequences:

- **A module activation is an N-tenant operation.** Enabling a module on a platform with 40 tenants
  means 40 migration runs. This is slow, it can partially fail, and it can hold locks on databases
  that are actively serving traffic. Under total self-hosting, the developer is the one waiting.
- **New tenants need the module's migrations applied at provisioning.** A tenant created after a
  module was activated must receive every currently-active module's schema changes. Provisioning is
  no longer "create an empty database" — it is "create a database and bring it to the current
  module set." This is a real change to the tenant-provisioning requirement and is not written down
  anywhere in the overview.
- **Partial failure — DECIDED: fail closed.** If migration fails for any tenant, **the module is
  not activated** and the failure is reported. There is no degraded state in which some tenants run
  the module and others do not.
- **Dry-run before activation — DECIDED.** Activation is a two-phase operation: a **validation pass
  that checks everything first**, and only if that passes, the actual run. *Requirements:* the
  dry-run result must be shown to the developer before the commit phase begins; the two phases must
  not be conflatable in the UI or the API; and a failed validation must change nothing.
- **On failure, roll back — DECIDED.** Migrations that already succeeded are reverted, so activation
  either completes fully or leaves no trace. *Requirements this generates:*
  - **Down-migrations become mandatory, not optional.** A module cannot be activatable unless it can
    be deactivated cleanly. This is an obligation on module authors and belongs in the module
    contract — the scaffolding CLI should refuse to generate a module without one.
  - **Rollback itself can fail.** A down-migration that throws mid-sequence can leave a tenant
    schema in a state neither the forward nor the reverse migration produced. **DECIDED: surface it
    loudly and require manual intervention** — the platform does not attempt to guess a repair.
    *Under single-tenant (D-8) "that tenant" is this deployment; the behaviour is unchanged.*
  - **Rollback must be per-tenant and resumable.** With N tenants, reverting means N down-runs; a
    failure partway through must not leave the developer unable to determine which tenants are
    clean.
  - **Activation failure must be diagnosable from the UI**, without reading server logs — which
    tenants succeeded, which failed, at which migration, and what to do next. Under total
    self-hosting the developer may not have easy log access, and "module not activated" alone is a
    dead end.
- **Dry run reduces failures; it does not eliminate them — say so in the docs.** Validation catches
  what is predictable: manifest well-formedness, dependencies satisfied, core-version compatibility,
  migrations parseable, connectivity and disk reachable. It does **not** catch anything dependent on
  real data or real timing — a migration that fails only against tenant 24's actual contents, or
  lock contention under production load. **Fail-closed behaviour is what makes a failure safe, not
  the dry run**; that is the property the design must guarantee.
- **Version skew across tenants.** If a module upgrade migrates 39 of 40 tenants and the developer
  then upgrades the core, those tenants are on different module versions simultaneously. Anything
  that assumes a uniform module version across tenants has to cope with that.
- **A module's migrations are arbitrary SQL from third-party code**, executed against every tenant
  database with the platform's credentials. A module can drop a core table. This sharpens O-4: the
  trust model must cover what a module's *migrations* may do, not just what its runtime code may do.
  **DECIDED: accepted, consistent with [O-4](open-gaps.md#L469).** A module's migrations run with the
  platform's credentials and are not scope-checked. *This is the same trust boundary as runtime code —
  a module that is trusted to run in-process is trusted to run SQL — and splitting it would require
  per-module database roles, which contradicts modules sharing one database and one service layer.
  Recorded as an accepted consequence, not an oversight.*
- **Rollback is a module-version problem.** Covered by the rollback decision above — down-migrations
  are now mandatory. Module *downgrade* (newer version → older) additionally needs either a working
  down-path across the version gap or an explicit forward-fix policy, since the manifest may declare
  compatibility ranges rather than exact versions. **DECIDED by [O-14](open-gaps.md#L1168): downgrade
  across a major boundary is refused — a module must be compatible with the running Vaah major to be
  activated. Downgrade *within* a major line is permitted where a working down-path exists;
  otherwise the operator applies an explicit forward-fix.**
- **Migration safety policy (L431 of the overview) must bind modules too.** The expand/contract
  discipline the platform holds itself to has to be an obligation the platform *enforces* on module
  migrations, or third-party authors will ignore it and the safety record becomes fiction.

**Lifecycle operations — three distinct actions, DECIDED:**

| Action | Effect |
|---|---|
| **Deactivate** | Deregister everything the module contributes — routes, services, views, models. The module remains installed. Modules that depend on it are deactivated with it at the next boot (O-1 deactivation cascade). |
| **Delete** | Remove the module itself. |
| **Reset** | Remove all of the module's migrations and its associated data — media, taxonomies, and the rest — returning it to as-installed state. |

**These are genuinely three operations, not one with a flag, and the distinction should be explicit
in the UI and the API.** Delete and Reset sound interchangeable and are not: Delete removes the
*code*, Reset removes the *data* and schema. A module can be deleted while deactivated, keeping its
data intact for later reinstall — which is the useful case for reinstalling a newer version without
losing tenant content.

**Requirements and consequences:**
- **Deactivation is restart-scoped, not immediate.** Installation state is resolved at startup, so
  deactivation means *the module is excluded from the next boot*. The UI must say so, and must not
  imply the change is live. (Consistent with the restart-based lifecycle already established in O-1.)

  This is a chosen constraint, not a framework limit — an Express router stays mutable while serving
  and live route removal is achievable. Restart-scoping is preferred because it makes the active set
  a pure function of data read at boot: there is no half-applied state, and revocation is guaranteed
  rather than best-effort.
- **Deletion must be refused while active**, or must deactivate first. Deleting a running module's
  code is not a state the platform should allow.
- **Deletion must be refused while another active module declares a dependency on it.** The manifest's
  `requires` list makes this checkable; this is a concrete use for it.
- **Reset is destructive and irreversible**, and it deletes **user data, not just platform-created
  data** — media and taxonomies are tenant content. *Requirements:* an explicit confirmation step
  that states exactly what will be deleted and how much; a clear statement that it cannot be undone;
  and, under D-7, an offer to export or back up first. This is the most dangerous operation in the
  entire module system and should be treated as such.
- **Reset requires a clean deactivation first** — down-migrations against live routes and services is
  not a state anyone should be in.
- **Reset must not touch the tenant's own records** that merely reference the module's taxonomies.
  Which side of that line a deletion falls on is a real design question — deleting a taxonomy that
  tenant content is tagged with leaves orphaned references.

**Resolved on this item:**
- ~~Which **lifecycle events** are emitted and who may subscribe.~~ **RESOLVED by
  [O-16.3](open-gaps.md#L1519)**: one Redis-backed event bus, namespaced and typed, with declarative
  module subscriptions. Activate/deactivate/delete/reset are events on it, as are module-owned domain
  events.
- ~~**Seeds** are not migrations.~~ **DECIDED: idempotent, duplicate-checked.** A seed declares how it
  identifies its records so re-running updates rather than duplicates; re-activation re-runs seeds
  safely; sample seeds are separated from required seeds; sample-data cleanup is owned by **reset**.
  *This prevents the defect that only surfaces in a demo — a module deactivated and reactivated
  producing duplicate sample data.*
- **Per-tenant module activation** — ~~unanswered~~ **MOOT under [D-8](open-gaps.md#L153)**: Vaah is
  single-tenant, so there is one deployment and one activation set. No subset question arises.

### O-6 · Queue, jobs & durable execution — **DECIDED**

**Decisions:**
1. **Jobs ship in v1. Durable workflows do not.** The Workflows/Automation surface in the overview
   remains a stated goal, but the durable-execution substrate underneath it is deferred. *This does
   not weaken the Workflows section — it sequences it. Build the semantics on an in-database
   workflow record first; choose the engine when multi-day, compensating processes actually need
   one.*
2. **Job substrate: Redis + BullMQ 6.3.12.** Chosen on efficiency, throughput, and being the
   industry-standard choice.

**Correcting an earlier recommendation.** I had leaned toward Postgres-backed queues
(pg-boss / graphile-worker) partly on the argument that it saves a container. That argument was
weak, because **Redis is already required by D-4's SSE design** — the marginal container saving of
dropping Redis for Postgres is zero. Once that premise is removed, BullMQ wins outright on the
stated criteria: it is the industry standard, it is genuinely faster for short units of work, and its
ecosystem (retries, rate limiting, flows, a mature dashboard) is years ahead of the Postgres options.
*I had also raised "job state commits with domain state" as a Postgres advantage — that remains true,
but the **transactional outbox below** solves the same problem for any substrate, so it was not
actually a decisive factor.*

**Requirements this generates:**
- **Transactional outbox — REQUIRED regardless of substrate.** Since events drive workflow triggers
  and job dispatch, committing domain state and then publishing produces phantom triggers when the
  process dies between the two writes. Write the intent row **in the same transaction** as the domain
  change; publish from the outbox afterwards. This also delivers the Postgres advantage above without
  committing to a Postgres queue. *This is the single most-skipped correctness measure in systems of
  this shape and gets much harder to retrofit — it belongs in v1.*
- **Jobs are at-least-once, and handlers must be idempotent.** At-least-once delivery is the only
  honest guarantee a queue of this kind offers. State the requirement on the developer-facing side:
  job handlers must tolerate being handed the same job twice. (The SSE transport in D-4 is
  at-most-once and separately requires idempotent refetch — the two align by design.)
- **Dead-letter handling and a visible failed-jobs surface.** A job that fails permanently must land
  somewhere the developer can see and retry, not disappear into a retry loop. Under D-7, visibility
  *is* the feature.
- **Concurrency limits per job type.** A self-hosted developer on one box must not have a report job
  starve email delivery.
- **Module activation migrations as queued work (DECIDED follow-on).** Activation runs long and is
  per-deployment (D-8), so it runs as a **job with visible progress**, not synchronously inside the
  activation request. The developer gets something to watch — which matters more than it sounds when
  activation can take minutes on modest hardware. *This connects O-6 to O-5b: activation is
  fail-closed, so a queued migration job must report its outcome back into the activation state
  machine rather than leaving the developer guessing.*
- **The outbox itself needs a relay** — something that reads the outbox and publishes. Under D-7 this
  must be part of the app process or a worker, not another container.

**Deferred to whenever workflows actually ship:**
- Temporal 1.24.0 (proven, heavy — wants its own cluster, a real D-7 cost) vs
  DBOS 5.2.11 (durable workflows as rows in the existing Postgres, light, young).
- The visual workflow builder ([L166-193](overview.md#L166)) — a large surface in its own right,
  which O-15 also flags.

*Note: the timezone point below no longer follows from D-8. A single deployment serves one business,
so its users may still span regions — but "per-tenant scheduling" is now "per-deployment scheduling
against the configured business timezone." Weaker than it was, still real.*

**Two distinct problems, often conflated — and they have different answers:**

- **Jobs** — single-unit background work: send an email, transcode a video, generate a report. Needs
  retry, backoff, concurrency limits, and a dead-letter path. **BullMQ 6.3.12** is the mature,
  well-supported choice and we already have Redis in the stack.
- **Workflows** — multi-step, multi-day, compensating processes. The decisive property: **a workflow
  instance should be a row in the application's own database**, so the order and its fulfilment
  commit together. A plain job queue cannot do this — saving an order and enqueueing its job are two
  writes, and a crash between them loses one.

**Requirements this generates:**
- **Transactional outbox.** Since the event bus drives workflow triggers, committing-then-publishing
  without one produces phantom triggers. Write the intent row in the same transaction as the domain
  change; publish from the outbox afterwards. *This is not optional if workflows are triggered by
  domain events, and it is the single most-skipped correctness measure in systems of this shape.*
- **Module lifecycle events feed this** (see O-5b). Activation must run migrations; a migration is
  long-running and per-deployment. Whether it runs synchronously in the activation request or as a
  queued job with progress reporting is a real decision — queued is kinder to a self-hosted
  developer on a small box, and gives them visible progress.

**Options for durable workflows — current state:**
- **Temporal** (`@temporalio/worker@1.24.0`) — the most complete and battle-tested. Also the heaviest:
  it wants its own cluster (or Temporal Cloud), which under **D-7 is a second major container** and
  substantially raises the operational burden for a self-hosted developer.
- **DBOS Transact** (`@dbos-inc/dbos-sdk@5.2.11`) — durable workflows **as rows in the application's
  own Postgres**, which is exactly the property above, with no extra infrastructure. Very recent.
- **pg-boss 12.37.1** — Postgres-backed job queue. Single-unit jobs only; no multi-step durable
  execution.
- **graphile-worker 0.18.0** — Postgres-backed job queue, strong SQL ergonomics. Same limit.
- *Earlier note recorded "first-party options are all weeks old and pre-1.0" — that no longer holds;
  both Temporal and DBOS are at 1.x and 5.x respectively. The bet is different now: Temporal is a
  proven-but-heavy dependency, DBOS is lightweight-but-young.*

**The D-7 tension, stated plainly.** Workflow durability has a cost, and under total self-hosting the
developer pays it in containers to run and back up. Temporal's own deployment guidance assumes a
cluster. That makes it a poor fit for "self-host everything, including all containers" unless the
developer accepts that operational cost.

**Decisions needed:**
1. **Do workflows ship in v1, or only jobs?** The overview has a substantial Workflows/Automation
   section ([L166-193](overview.md#L166)) including a visual builder — so this is not a question the
   doc has answered. But the *durable execution* substrate underneath it is a separate, heavier call.
2. **Jobs substrate — Redis or Postgres?** Redis is already in the stack and BullMQ is mature. But
   Postgres-backed means one fewer container and, more importantly, **job state commits with
   domain state** — removing the two-write problem for the simple case too.
3. **If workflows ship — Temporal or DBOS?** Temporal is proven and heavy; DBOS is light and young.

### O-7 · Testing & quality engineering — **DECIDED (core gates); module gates not Vaah's responsibility**

**Scope of responsibility — DECIDED, and it corrects an earlier proposal.** I proposed a
contract-completeness gate covering all modules. That overreached: **Vaah is not responsible for what
third-party module authors ship.** A module author who omits schema declarations has written a bug in
*their* module, and making it Vaah's problem would be both unenforceable and wrong.

The correct division:

- **Vaah's responsibility** — the **core's own contract is complete and verified.** The core's routes,
  schemas, and agent tools are Vaah's to guarantee. This gate applies to the core and is
  non-negotiable, because agents depend on it.
- **The module contract is defined and documented, but conformance is the author's obligation.**
  Vaah's role is to *make conformance easy and stated*: the module contract specifies what a
  conforming module does, the scaffolding CLI generates conforming modules by default, and the
  developer-facing guidance says so. Beyond that, a non-conforming module is a bug in that module.
  *This is consistent with every other decision in O-1/O-4: the platform validates what it owns and
  defines the contract for what it doesn't.*

**Requirements:**
- **Core contract completeness gate** — a check that fails the build when a core endpoint exists in
  the contract without declared parameters or response shape. This is the highest-value gate in the
  list, because agents depend on it and the failure is invisible when it happens.
- **OpenAPI backward-compatibility gate in CI.** The overview promises "backward-compatible changes
  only, breaking changes get a new version" ([L334](overview.md#L334)). That promise is currently
  decorative — nothing checks it. An automated contract diff between releases makes it real. Cheap,
  and it protects the most-referenced guarantee in the API section.
- **Expand/contract enforcement for core migrations.** O-5b made down-paths mandatory for modules; the
  core holds itself to the same discipline and should verify it rather than assert it.
- **Permission-path coverage as an explicit gate.** Field-level permissions are checked at code level
  before operations — the highest-value thing to test and the easiest to under-test. Coverage
  percentage is a poor proxy here; an untested authorization branch can sit inside well-covered code.
- **In-process Postgres for tests**, so DB-dependent tests need no external infrastructure — directly
  serving the D-7 self-hosting constraint and the overview's one-command local dev loop.
- **Module test harness** remains a developer-facing capability ([L45](overview.md#L45)) — provided as
  tooling, not as a guarantee the platform makes about the module.

**Resolved — the smaller items:**
- Whether visual regression and accessibility checks are blocking gates or warnings. Both are listed
  as aspirations in the overview; as warnings they mean little, as gates they can block a developer on
  a subjective diff.
- ~~**Who owns the version-skew harness.**~~ **RESOLVED by [O-3](open-gaps.md#L369)**: the module
  author runs it in the module's own CI. Vaah runs sanity checks at activation (registration only)
  and does not run module tests.

### O-8 · CI/CD & release engineering — **DECIDED**

The load-bearing item here is **expand/contract migrations**, not the pipeline. The lifecycle runs
migrations on activate/upgrade ([O-5b](open-gaps.md#L498)), so without expand/contract a rollback
after a migration is data loss — directly contradicting our own DR promise that upgrades are
"reversible where possible to prevent data loss" ([L436](overview.md#L436)).

**Decisions:**

1. **Expand/contract is mandatory for core and for the scaffolding CLI's generated migrations.** No
   breaking schema change ships in a single release. Two-release shape:
   - *Expand* — add the new column/table/index alongside the old one; write to both; old code still
     reads the old shape. Backward compatible.
   - *Contract* — a later release removes the old one, after the expand release is proven in the
     field.
   - **Corollary: down-migrations are not always possible for a contract step** — once old data is
     gone, rollback means rolling *forward*. This must be stated plainly rather than left as a
     surprise during an incident. *This refines, and does not contradict, [O-5b](open-gaps.md#L498):
   down-paths are mandatory where they can exist, and their absence must be declared in the
   migration itself rather than discovered.*
2. **Trunk-based development, short-lived branches.** Long-lived feature branches guarantee merge
   conflicts and stale specs. The spec-driven flow ([L42](overview.md#L42)) produces auditable
   commits, which suits a single trunk.
3. **Required checks on every PR**: lint, type-check, unit + integration tests, migration validation
   (up **and** down where applicable, with expand/contract enforcement), dependency SCA
   ([L383](overview.md#L383)), OpenAPI backward-compatibility, and the accessibility scan floor
   ([O-12](open-gaps.md#L920)).
4. **OpenAPI backward-compatibility is enforced in CI**, alongside core contract completeness
   ([O-7](open-gaps.md#L712)). *This is the gate that survived the user's O-7 correction — modules are
   not gated, but the core's own contract is.*
5. **SBOM and signed release artifacts.** The module lifecycle loads code at runtime ([O-1](open-gaps.md#L202)),
   so artifact provenance is a supply-chain control rather than a nicety. SLSA provenance where
   practical.
6. **Release: semantic versioning** on the contract boundary agreed in [O-14](open-gaps.md#L1026) —
   the major version **is** the compatibility boundary modules declare against. Minor and patch
   releases never break the extension contract.
7. **LTS policy**: the current major and previous major are supported; security fixes for the
   previous major for a defined window. *Recorded as a policy shape — the specific window length is a
   maintainer commitment rather than a technical requirement, and is left to the project rather than
   fixed here.*

**Not decided, deliberately:** which CI system. It is an implementation choice with no requirements
impact, and naming one here would be the requirements doc legislating tooling.

### O-9 · Agent evaluation strategy — **DECIDED**

Guardrails and telemetry tell us *what an agent did*. Nothing tells us *whether it did it well*. For
most software, tests assert a correct result; an agent's output is non-deterministic, so the same
prompt yields a different answer run to run and "did this change break it?" has no obvious answer.
Today the honest answer is *you find out in production*.

**Two decisions make this load-bearing:**
- **Agents are first-class actors with real permissions** ([L56](overview.md#L56)) — they can disable
  users, change settings, delete records. An eval gate is not quality hygiene; it is the control
  that stops a prompt change from quietly widening what agents may do.
- **Model routing with automatic fallback** ([L296](overview.md#L296)) means a provider shipping a
  new model version changes behaviour **with no code change and no deploy** — an invisible production
  change.

**Decisions:**

1. **Versioned offline eval suite**, runnable without a deploy, in two parts:
   - **Golden tasks** — fixed scenarios with expected *outcomes* ("given these 20 records and this
     instruction, the agent filters by status and touches nothing else"). The regression net.
   - **Red-team set** — prompt injection, data exfiltration, permission escalation. The guardrails
     ([L297](overview.md#L297)) *promise* these defences; a red-team set is what proves the promise
     still holds after a change.
2. **Assertions are invariants, not exact outputs.** *This is the honest constraint, and it must be
   stated rather than papered over:* a non-deterministic agent cannot be asserted against a golden
   string. Evals assert that the right records changed, that the wrong ones did not, that no
   permission was escalated, and that guardrails held. This is weaker than a unit test and is the
   strongest thing available.
3. **Evals gate changes that can alter behaviour**: prompts, tool definitions, and model configuration
   are blocking. Everything else is advisory.
   *Recorded risk, because it is how eval gates die:* a noisy suite that blocks merges trains people
   to bypass it, which is worse than no suite. Gating therefore requires the suite to be stable and
   fast; a flaky gate must be fixed or demoted, not tolerated.
4. **Model versions are pinned and recorded per agent run**, so a regression is diagnosable at all.
   *Without this, "it got worse sometime" is undebuggable — we cannot tell which model handled which
   request.* Recording the resolved model alongside each trace is the prerequisite for the whole
   capability.
5. **Fallback behaviour is a per-deployment configuration**, not a hardcoded default. Automatic
   fallback to a secondary provider ([L296](overview.md#L296)) is *good* under a provider outage and
   *bad* when it silently changes behaviour on a provider regression. Which of those an operator
   wants depends on their deployment, so the choice is theirs.
   *This answers the "who owns a provider regression" question pragmatically:* with pinning and
   recording, it is always attributable — and the operator who chose the BYO key owns the provider
   relationship ([O-13](open-gaps.md#L965)) while Vaah owns the routing and recording behaviour.
   Responsibility is knowable rather than disputable.*
6. **Core owns the eval framework; the CLI scaffolds eval files for modules.** *Consistent with the
   O-7 and O-12 posture:* Vaah defines the contract and generates the starting point, and does not
   guarantee or gate what a module does with it.

**Deliberately not decided:** the eval framework implementation, and which LLM judges the suites.
Both are implementation choices with no requirements impact, and the second has an obvious self-check
problem — an LLM grading another LLM's work needs its own eval, and recording that dependency honestly
is left to the design.

### O-10 · Observability standard & audit integrity — **DECIDED**

Observability is already well specified at [L374–390](overview.md#L374-L390). The gap was **audit
integrity**: the doc says *what* is logged and never says anything about the log being *trustworthy*.

**Decisions:**

1. **Audit is an ordinary application table, not a tamper-evident store.** No hash chaining, no
   separate append-only role, no WORM-style enforcement. Entries are written by the same service
   layer as everything else, and anyone with database access can alter or delete them.
   *Consequence, recorded deliberately:* an audit trail is a **diagnostic and operational record,
   not evidence**. It supports "what happened and when" investigations and reconstructs agent
   activity; it does **not** support "prove this to a regulator, an auditor, or in a dispute",
   because it is not tamper-proof. Enterprise buyers who require evidentiary-grade audit logs are
   out of scope for v1. *This is the accepted cost of not adding the machinery — recorded so it is
   a position rather than an oversight.*
2. **Right-to-erasure does not reach the audit log.** Data deletion removes a person's records from
   the application's data model; the audit entry recording what was changed stays, including the
   actor and the nature of the change. Erasure and audit are deliberately not reconciled.
   *Consequence:* personal data persists in the audit log after a deletion request. Anyone operating
   Vaah in a jurisdiction with GDPR or equivalent obligations must scope this in their own policy
   — typically a retention period applied to the audit table separately from the data-deletion flow.
   *Accepted for v1, per the same simplicity reasoning as D-8 and the GraphQL drop: the alternative
   (pseudonymisation plus tombstones plus chain integrity) is a large amount of machinery for a
   capability most self-hosted deployments will not exercise.*
3. **Audit logging fails open, always.** If the audit write fails — locked table, disk full, timeout
   — the **business operation proceeds** and the failure is logged as an error through the normal
   observability facility, raising an operational alert. Nothing is refused and nothing is retried
   into a blocking path.
   *Consequence:* under sustained storage pressure the audit trail develops gaps, and a
   security-relevant action may complete unrecorded. This is the deliberate trade: **availability
   over completeness.** The alternative — refusing actions whose audit entry cannot be written —
   means a full disk takes the platform offline, converting a degraded audit trail into a full
   outage. For a self-hosted platform the operator already owns that disk, so the platform refusing
   to operate is the worse outcome.
4. **Audit entries record what changed, not merely that something changed.** "Settings updated" is
   near-worthless in an investigation; "LLM provider changed from X to Y, key rotated" is what an
   operator needs. Before/after diff for the affected fields is required, with redaction of secret
   material (compare presence and fingerprint, never the value). Highest value for agent actions,
   where attribution without resulting state leaves no way to assess impact — this directly extends
   the existing dual-attribution entry at [L126](overview.md#L126).
5. **OpenTelemetry is the tracing standard**, with the correlation IDs already promised at
   [L377](overview.md#L377) carried as its trace/span identifiers rather than a parallel
   invention. Traces cover the service layer, agent tool calls (where an agent's trace is the parent
   of the HTTP call it made) and workflows — so a single agent action is one trace spanning
   model call, tool call, and resulting DB writes.
6. **SLOs and error budgets are a stated target, not assumed.** Structured logging already specified.
7. **Health checks cover the actual dependency set**: PostgreSQL, Redis (B1 queue + sessions +
   pub/sub behind one health probe), and the media storage backend. Under D-8 single-tenant there is
   no per-tenant dimension to these checks — one deployment, one health surface.

**Reads are audited too — but at a coarser grain.** Reporting queries run through the service layer
(O-15) and are therefore auditable operations. A row-level read is not logged (the audit store would
outgrow its own purpose), but *who ran which report, with what filters, when, and what row count came
back* is recorded. This closes a real gap: bulk data extraction is the highest-value abuse case, and
without read auditing it is invisible.

**What this decision explicitly gives up**, so it is not rediscovered later as a surprise:

- **Evidentiary-grade audit.** Vaah's audit log cannot survive a hostile or compromised database.
  Any requirement for tamper-proofing — hash chaining, WORM storage, external append-only sink —
  would be a new feature against a different data model, not a toggle on this one.
- **Audit-scoped erasure.** Data deletion and audit deletion are separate lifecycles. A deployment
  operating under GDPR must apply its own retention policy to the audit table; Vaah does not enforce
  or cascade one. *This is the one consequence with external compliance weight, and O-13 owns the
  question of what Vaah says publicly about its scope.*
- **Guaranteed-complete trails during incidents.** By design, the audit log is lossy exactly when
  the system is under stress. The compensating control is the operational alert from decision 3 —
  a gap must be *visible*, not prevented.

**Left to O-11**: fail-open audit logging means an audit-write failure does not block the business
operation, so the only signal is an error log and an alert. O-11 owns alerting behaviour during a
dependency incident, including whether a sustained run of audit failures escalates beyond a
notification.

### O-11 · Resilience & failure modes — **DECIDED**

The Disaster Recovery section ([overview.md:428](overview.md#L428)) already covers failover, backups,
RPO/RTO, and graceful degradation. O-11 is behaviour under failure *inside a running request*.

**Decisions:**

1. **Saga / compensation dropped entirely — not deferred.** A large share of why resilience is hard
   is multiple things that can disagree: multiple databases, multiple services, partial commits
   across them. Saga and compensation exist almost entirely to paper over that. Under D-8 Vaah is
   single-tenant on **one database**, so a change spanning several tables is one transaction, not a
   distributed one. Multi-step operations become either a single DB transaction (correct, simple) or
   a workflow with explicit steps ([L181](overview.md#L181)). *This is the largest simplification in
   O-11 and a direct dividend of the single-tenant decision — it is a genuine deletion, not a
   deferral. The patterns return only if multiple databases come back, which D-8 rules out.*
2. **Timeout on every outbound call, no exceptions.** No timeout means a hung dependency holds a
   request open indefinitely and eventually exhausts the connection pool. Slower dependencies get
   longer budgets — a BYO LLM key on someone else's infrastructure is the one component Vaah cannot
   control.
3. **Retry policy is declared per job, Laravel-style — not platform-wide.** Modelled directly on the
   `laravel/framework` job API, which is the right precedent: each job declares its own `tries`,
   backoff, and timeout, and retries are **opt-in per job, not automatic**. Defaults: no retry unless
   the job asks for it; exponential backoff with jitter on retry; failed jobs land in the dead-letter
   view ([O-6](open-gaps.md#L609)) with the reason.
   *This resolves the tension flagged against O-7:* the burden lands on the job author, which is
   exactly the position Laravel takes and exactly the position Vaah already took on module
   conformance — **Vaah defines the contract and provides the safe default, and does not guarantee
   what a module or job does with it.** A job author who fails to declare retry-safety gets a
   non-retried job, not a silent duplicate.
   **Safety net:** a job that fails and was *not* eligible for retry is surfaced in the dead-letter
   view with an explicit "not retried — retry-safety not declared" note. Nothing vanishes quietly.
   *Idempotency for jobs with outside effects (sending email, calling a payment API, posting a
   webhook) remains the author's responsibility, with the outbox ([O-6](open-gaps.md#L609)) and the
   idempotency key ([L349](overview.md#L349)) available as the supported mechanism.*
4. **Circuit breakers on external services only — never on the database.** A breaker on Postgres
   would mean *stopping using the database*, which makes an outage worse rather than better.
   Breakers apply where a fallback exists (cached value, skipped enrichment) or where an upstream is
   rate-limiting us — LLM providers, payment gateways, outbound webhooks. **Exception:** Redis *is*
   breaker-eligible for session reads, because DR already promises "cache misses fall through to the
   DB" ([L434](overview.md#L434)) — there is a real fallback behind that breaker.
5. **Distributed locking for scheduled and single-flight jobs.** Required by the stateless
   horizontally-scaling app tier ([L431](overview.md#L431)): two instances can otherwise fire the same
   cron at the same instant and run scheduled reports or cleanup twice. Redis lock (already a
   dependency) with a bounded TTL, so a crashed holder cannot deadlock the schedule permanently.
   Mirrors Laravel's `WithoutOverlapping`.
6. **No unbounded work inside a database transaction — new requirement.** *This is the failure mode
   most likely to actually take the platform down, and it originates in the agentic features.* If a
   transaction is open and the code inside it makes an LLM call, a slow provider holds a **database
   connection and row locks** for the whole call duration. Multiply by concurrent agents and the
   connection pool is exhausted — at which point *everything* fails, including the health checks, so
   the platform presents as dead rather than degraded.
   **Requirement:** external calls happen **outside** transactions; the transaction opens only around
   the final write. Work inside a transaction is always bounded.
7. **Dead-letter handling and queue backpressure** — already decided in [O-6](open-gaps.md#L609);
   no change here.
8. **Bulkheads** are subsumed by decision 4: a breaker plus a bounded worker pool is what keeps one
   failing dependency from consuming all capacity.

**Resolved — the timeout value is now decided:**

- **Default 5 minutes** for the agent-facing LLM ceiling, configurable per deployment. The *split*
  (generous for agent calls, short for everything else) was already decided; the value is a middle
  choice — long enough for a substantial agent run with several tool calls, short enough that a
  hung provider is abandoned before an operator concludes the platform has frozen.
  *A default, not a constraint: an on-premise deployment doing long batch-style agent runs needs a
  different ceiling from an interactive chat product, which is why it is configurable. **The split
  between the two tiers matters more than either number.***

**Note on O-10 interaction:** fail-open audit ([O-10](open-gaps.md#L772) decision 3) and these
timeouts compound — during a dependency incident the platform degrades rather than blocks, and the
audit trail develops gaps exactly then. The compensating control is alerting (O-10 leaves escalation
of sustained audit-write failure to O-11), plus decision 3's dead-letter visibility.

### O-12 · Accessibility conformance target — **DECIDED**

Currently one line at [L304](overview.md#L304) ("WCAG-conformant, keyboard-navigable UI") — a
slogan rather than a requirement. Nobody can check it, and "WCAG-conformant" without a version and a
level is not testable.

**Why this is elevated rather than left as a UI bullet:** accessibility is a **procurement gate in
enterprise deals** — asked in security questionnaires and vendor reviews. It sits beside Security,
not under a UX heading. Given the Platform Standing section targets enterprise adoption
([L438](overview.md#L438)), a vague requirement here blocks deals.

**Decisions:**

1. **Target: WCAG 2.2 AA.** 2.2 is the current version (2.1 is the older widely-quoted one); AA is
   the standard procurement bar. AAA is not realistic for a platform shipping at this speed.
2. **Contrast enforced as design token constraints.** *This is load-bearing, not cosmetic.* O-2 made
   partial design-token override a v1 decision, which means third-party modules can override colour
   tokens. Contrast ratios therefore become **token-level constraints**, so a module overriding a
   colour token cannot produce unreadable text. *This is the only mechanism by which contrast
   survives third-party modules, and it is what ties O-12 to O-2 rather than leaving them
   independent bullets.*
3. **Keyboard and focus are platform conventions, not per-module choices.** Tab order, focus
   trapping in modals, focus restoration on close, skip-to-content — these must be platform rules
   with module obligations. If each module decides its own focus behaviour the assembled UI is
   inconsistent and unusable, and the inconsistency is invisible until someone tries to use it.
4. **Automated scanning in CI is mandatory, and documented as a floor.** Scanners catch roughly
   **30–40% of WCAG issues**; everything they miss — screen-reader semantics, focus management,
   keyboard traps, error announcement, custom-component contrast — is the expensive part.
   *Stating the target without this produces a false assurance, which is worse than none: it stops
   the question being asked. The remaining two-thirds is owned by design review and manual testing.*
5. **Reduced-motion supported**, and reduced-transparency where applicable.
6. **A module that fails accessibility conformance: Vaah does nothing** — no block, no flag, no
   gate. *Consistent with the O-7 position: Vaah defines the contract and the CLI ships accessible
   output by default, but does not take responsibility for what a module does with it.*
   **Recorded consequence:** an enterprise buyer who installs a non-conformant third-party module
   gets **no protection from Vaah** and inherits the liability. Mitigation available to the operator
   is the module's own accessibility statement, so **module manifests should carry an optional
   accessibility conformance field** — published information for the buyer, not an enforced gate.
   *This is the deliberate cost, chosen for consistency rather than overlooked: the alternative would
   make Vaah responsible for third-party code it cannot fix.*

**Left to O-4:** the same "flag vs. block" question recurs for marketplace curation. O-12's answer
does not pre-empt it, but it sets the precedent — Vaah's default posture toward module quality is
**inform, not enforce**.

### O-13 · Data privacy & compliance scope — **DECIDED (with one conflict flagged against O-10)**

Two earlier decisions create privacy exposure, both reasonable at the time: erasure does not touch
the audit log, and the audit log is an ordinary editable table ([O-10](open-gaps.md#L772)). Neither is
wrong — but O-13 is where the platform's scope decisions become someone else's legal problem.

**Decisions:**

1. **Vaah is GDPR-ready and owns the privacy mechanisms in core.** Vaah manages **consent storage**
   and **data deletion** as platform capabilities, available to core and to modules. This is a
   stronger position than "provide mechanisms and let the operator own policy" — Vaah carries the
   machinery and the correctness of it.
   *Scope of the claim must be defined, because "GDPR-ready" is not self-defining:* it means Vaah
   provides working consent capture and storage, a data-deletion workflow that actually finds and
   removes a subject's data across core **and module** data, retention configuration, and export. It
   does **not** mean Vaah is the data controller — under single-tenant self-hosting the operator is
   the controller and the lawful basis is theirs to establish. *Recorded explicitly because an
   unqualified "GDPR-ready" badge on a self-hosted platform would be a misleading claim.*
2. **Data residency: Vaah is not responsible.** Not a core feature, not a first-class section. Under
   D-8 single-tenant residency is largely settled by architecture — one deployment, one database,
   wherever the operator put it, and D-7's total self-hosting means Vaah does not phone home. The
   operator's deployment location is their decision and their obligation.
   *Note:* this does not remove the media/object storage question, which the operator configures.
3. **Sub-processors and AI-provider obligations: Vaah is not responsible.** Decided on the strength of
   an earlier decision rather than in isolation: **BYO LLM keys** ([L295](overview.md#L295)) means the
   provider relationship and its data-processing terms are the **operator's**, not Vaah's. Vaah ships
   no model of its own and processes no data on anyone's behalf, so it operates as a sub-processor to
   no one. This substantially reduces Vaah's exposure and is the reason this item can be dropped
   rather than specced. *It is also load-bearing: if Vaah ever adds a first-party or proxied model
   path, this decision must be revisited.*
4. **Consent management is a core capability.** Vaah stores consent state and enforces it — a
   consent record per subject, versioned consent text, withdrawal at least as easy as grant, and
   consent enforced as a gate on data processing rather than merely recorded. *The distinction is the
   requirement: a consent log nobody reads is decoration, and a withdrawal that leaves the grant in
   place is worse than no consent system at all.*
5. **Data deletion is a core workflow**, with a documented subject-right process (access, export,
   deletion, rectification) that reaches core and module data through the shared service layer.

**🔴 Conflict resolved — "GDPR-ready" vs. the O-10 audit decision.** Raised here because O-13's
GDPR-ready position and O-10's audit decision were in direct tension, and it was not a documentation
nicety:

- O-10 decided **erasure does not reach the audit log** — personal data persists after a deletion
  request — and that the log is an ordinary editable table.
- O-13 decided Vaah is **GDPR-ready** and manages **data deletion** in core.

An audit log retaining personal data after an erasure request is, on its face, a gap in the deletion
workflow O-13 promises. Options considered: a retention period on the audit table; pseudonymising the
subject's fields on erasure; or excluding audit data from the GDPR claim and saying so plainly.

**Decision: a retention period on the audit table**, applied independently of the data-deletion
workflow. Audit entries are purged after a configurable period; the operator sets it. Pseudonymisation
was not adopted — it would reverse the simplicity trade made deliberately in O-10. Excluding audit from
the claim was not adopted either, because the retention period substantially answers the concern.

**Consequence, recorded rather than hidden:** for the duration of the retention window, personal data
still exists in the audit log after an erasure request. The claim Vaah can honestly make is therefore
**"GDPR-ready with a configurable audit retention period"** — not "GDPR-compliant", and not a claim
that erasure reaches the audit trail. *The compliance posture must state this limit explicitly; a
deployment in a strict jurisdiction sets a short retention period accordingly.*

### O-14 · Versioning & compatibility for the extension contract — **DECIDED**

The overview promises extension points are "stable across versions (semver)" ([L98](overview.md#L98))
and that "modules never reach into core internals." Three earlier decisions make that load-bearing:
modules **consume core design tokens** (O-2), **declare a compatibility range** in their manifest
(O-1), and **own migrations with mandatory down-paths** (O-5b).

**Decisions:**

1. **Two independent versions.** Vaah has its own version; each module has its own. They are not
   coupled to each other — a module's version is its own, and compatibility with Vaah is declared
   separately (decision 2).
2. **A module declares `vaah_compatibility` in its manifest**, as an explicit range — e.g.
   `"4.0.*"`. The key is named and stable; it is part of the module contract.
3. **A compatibility check with upgrade action.** The admin console offers a control to check installed
   modules against the running Vaah version and upgrade the incompatible ones accordingly.

**Consequence of decision 1 — the major version IS the contract boundary.** With a single Vaah
version rather than a separately-versioned contract, there is no independent contract line to protect.
So **major version boundaries become the only place the extension contract may break**: within a
major line (4.x), extension points must remain stable; a major bump (4→5) is where breaking changes
are permitted. *This should be stated explicitly in the docs* — it is the rule that makes the promise
in [L98](overview.md#L98) enforceable, and it is currently implied nowhere.
- Modules declaring `4.0.*` are supported across the whole 4.x line, including 4.9.
- A module needing a 5.0-only feature declares `5.0.*`, and will not load on 4.x.

**Requirements this generates:**
- **Enforce the declared range at load time — do not merely document it.** A module declaring a range
  the running Vaah does not satisfy must be **refused, with a clear message naming both versions**.
  Loading it and hoping is the failure mode this exists to prevent.
  *This continues the pattern already chosen three times: activation is fail-closed, contract
  conformance is fail-closed, startup failure is skip-and-disable. An unmet compatibility
  declaration is the case where failing closed is unambiguously right — the alternative is a module
  running against a core it was not written for, failing unpredictably and much later.*
- **The compatibility check is read-only.** It reports which modules are incompatible and why; it does
  not silently change anything. Upgrading is a separate, explicit, per-module action.
- **An incompatible module is handled like any other unmet module requirement** — it does not load,
  and is surfaced in admin health rather than failing the boot. Consistent with the O-4 startup rule.
- **What "upgrade accordingly" means is not yet defined** — the module author publishes a version
  compatible with the newer Vaah, and the developer installs it. Vaah does not upgrade third-party
  module code on the developer's behalf, because it does not own or control that code. *Needs a
  decision: does the button only check and report, or does it also fetch and install?*

**Under D-7 (total self-hosting), the check cannot phone home.** The overview already forbids
phoning home ([L408](overview.md#L408)). So "check module compatibility" must work against a
**newly-installed or upgraded local Vaah version** — the developer runs the new version, and the
console reports which of their modules are now incompatible. It is not a registry lookup. *This is
worth stating in the requirement, because "check for compatibility" otherwise reads like a network
call to a marketplace.*

**Consequences to carry into implementation (requirements recorded; the *inventory itself* is a
design deliverable, not a requirements decision):**
- **Contract inventory.** Anything not written down as part of the extension contract is not covered
  by the stability promise. If a module reaches into an internal service because it was convenient and
  worked, that becomes de facto API — and deprecating it later is a breaking change we cannot make.
  The extension surface needs to be *enumerated*, not discovered retrospectively by whatever the first
  module happened to use. *The [L86–98](overview.md#L86-L98) list is the v1 inventory; what remains is
  the design work of writing the contract for each entry, which is not a requirements question.*
- **Deprecation window.** Within a major line, a deprecated extension point survives until a **codemod
  or an explicit migration note** accompanies its removal. A time-based window without that backing
  would be aspirational rather than a promise — *so the requirement is the backing artefact, not a
  number of months.*
- **Version-skew CI** — resolved by [O-3](open-gaps.md#L369): the module author runs it in the
  module's own CI.

### O-15 · Unstated heavy dependencies

#### Reports / BI — DECIDED (largest item, was the primary scope risk)

**Decision: build it — a fastest-possible Metabase-inspired reporting surface, over the app's own
database, with both a manual and an AI-based query builder. Scope is platform data (core-owned
entities), not module data. Scheduled reports run via cron.**

**Why this is coherent despite being the largest item in the document.** The usual objection to a
query builder is that it becomes a side door into the schema, bypassing the service layer and
therefore RBAC and field-level permissions. Building on the app's own database **through the service
layer** removes that objection — reads inherit permission checks, audit, and attribution rather than
sidestepping them. This is the same architectural argument the document already makes for agents
and workflows ([L204-216](overview.md)).

**Requirements:**
- **A reporting abstraction in the core** — a query definition (source, fields, filters, grouping,
  aggregation) executed through the service layer, returning data plus typed metadata for charting.
  This is the internal shape both builders target; neither builder is a special case.
- **Two builder paths, one model.** A **manual builder** (visual: source, fields, filters, group-by)
  and an **AI-based builder** (the Reporting agent proposes a query; the user reviews and saves it).
  Both produce the same query-definition artifact. **The AI path is assistive, not autonomous** — it
  proposes, a human approves and saves. This matters because the Reporting agent already exists as a
  built-in agent ([L231](overview.md#L231)) and is the default for this section.
- **Permission enforcement is the design constraint, not an afterthought.** Every query resolves
  against the requesting user's field-level permissions ([L55](overview.md#L55)). A query the user may
  not run should be rejected at validation with a clear message — not silently filtered, which
  produces plausible-but-wrong numbers, and not executed then filtered, which leaks existence.
  *This is the highest-risk requirement in the item and the one most likely to be got wrong under
  delivery pressure.*
- **Scheduled reports via cron** — runs through the O-6 job queue (BullMQ), which supplies retry,
  dead-letter handling, and visibility. Under D-7 a developer must be able to see that a scheduled
  report failed; a silent failure here looks identical to "no data".
- **Module data is out of scope for v1** — platform data only. This keeps the reporting surface from
  requiring a manifest-declared extension point for module-contributed queries, which would grow the
  module contract again. *Recorded as a deliberate boundary; modules can still contribute dashboard
  widgets ([L87](overview.md#L87)) and consume reports ([L96](overview.md#L96)) without owning
  queries.*

**Honest cost, stated rather than hidden:** this is still a large build — a query engine, two
builders, a charting layer, a dashboard grid, and scheduling. "Fastest possible" is the right
instinct, and the sharpest way to honour it is to make the manual builder ship first and the AI
builder land on the same query model afterwards — it needs no separate engine.

#### Sessions, auth & push — DECIDED

**1. Sessions are Redis-backed — DECIDED.** Driven by [L56](overview.md#L56) ("if a user is marked
inactive or deleted, they **immediately** lose the ability to perform any action"). *Immediately* is
the requirement, and it is the whole argument:
- Redis-backed sessions make revocation a matter of destroying the session — immediate and trivial.
- JWTs are cryptographically valid until expiry and **cannot be revoked**. The usual mitigation is
  short access-token lifetime plus rotating refresh tokens, which approximates revocation at the cost
  of refresh rotation, revocation lists, and clock-skew handling — and still gets the semantics subtly
  wrong in exactly the security cases that matter.
- Redis is already required (O-6 queue, D-4 SSE), so this is a reuse rather than a new dependency.
- **D-8 strengthens this:** single-tenant removes the horizontal-scale pressure that makes stateless
  tokens attractive. Server-side session state is not a real cost for a one-deployment platform.

**2. Social login is out of the box — DECIDED.** Google, Facebook, GitHub, and comparable providers
work without module or code work.
- **OAuth 2.1 / OIDC** is the underlying protocol. Because the requirement is *social login
  out of the box*, the practical consequence is that **the core ships provider implementations**, not
  merely protocol support. Protocol support without shipped providers fails the stated requirement.
- **SAML is not committed to for v1.** It is XML-heavy with uneven library support, and no named
  requirement depends on it. *Recorded as deferred rather than excluded — enterprise customers needing
  SAML are exactly the kind of audience a module can serve.*
- *Requirement:* providers must be **configurable from the admin Settings page** without code changes,
  consistent with "social login" already appearing in Settings ([L134](overview.md#L134)) and with the
  self-hosting posture — a developer enabling Google login for their instance should not need code.

**3. Push notifications via Web Push / VAPID — DECIDED.** Browser-native, no third-party delivery
service, no vendor account required. FCM was rejected as a Google dependency, which sits badly with
D-7's no-phoning-home posture ([L408](overview.md#L408)).
- *Requirement:* key material is **self-hosted** — the developer generates and stores their own VAPID
  keys on their instance. No shared platform keys, consistent with self-hosting.

**Coupled requirement — agent credentials are NOT user sessions.** [L212](overview.md#L212) has agents
authenticating with "scoped credentials." That is a distinct credential type with its own revocation
semantics, audit attribution, and lifetime, and must not be implemented as a user session with an
agent label. *Stated explicitly so it does not become an implementation accident — agent
attribution is load-bearing across the audit and UI requirements.*

#### Email/notification templates — DECIDED

**Decision: a structured template format, plus a visual editor for humans. Template versioning is
NOT required.**

**The reasoning that produced this split.** [L157](overview.md#L157) asks for a visual editor "with
variables and versioning," and [L212](overview.md#L212) lets agents send email. Those two requirements
were written as one editor, but an agent cannot use a visual surface — so a single editor either
fails agents or drags them through a UI they don't navigate. Separating them fixes that:
- **Structured templates** are the format — the agent-native path, with variables and no editor
  involved. This is also what makes templates addressable by modules ([L160](overview.md#L160)).
- **A visual editor for humans** sits on top of the same format as an optional convenience, not a
  prerequisite for using templates.

**Requirements:**
- **The format is the contract, and it is versioned** — the *format* needs a version so a stored
  template can be rendered by a later core version. *This is separate from template versioning, which
  is explicitly not required.* Without a format version, upgrading the platform can silently change
  how historical templates render.
- **Variables are declared, not free-form.** A template declares which variables it uses and their
  types, so a missing variable at send time is a validation error rather than a blank in a customer's
  inbox. This matters more once agents compose templates — an agent should not be able to emit a
  template referencing a variable that does not exist.
- **Modules register their own templates and variable types** ([L160](overview.md#L160)), which
  means module-contributed templates declare variables the core knows nothing about. The declaration
  is what makes that safe.

**On dropping versioning — noting the one consequence, not re-litigating.** Without template
versioning, **there is no history of what a customer was sent.** If a template is edited and a
delivery dispute follows, the prior content is gone. This is a real trade — usually taken when the
alternative is manual snapshotting by the developer, which is acceptable at Vaah's scale and under
D-7 where the developer controls the instance. *Recorded so the choice is visibly a choice.*

#### i18n & image processing — DECIDED (abstraction decided; library names are implementation)

Both were the last routine dependencies. **The requirement in each case is the abstraction, not the
library** — so these can be decided at requirements altitude without pinning a package the project may
want to replace later.

**1. i18n — a standard i18n package behind a Vaah abstraction. DECIDED.**
- The mechanism is already settled ([L324](overview.md#L324)): UI strings externalised and translatable,
  modules reusing the same mechanism, content translation remaining each module's responsibility.
- **Modules never import the i18n library directly** — they go through Vaah's translation API. *This is
  the load-bearing part: it is what makes the library replaceable, and what guarantees a module's
  strings localise identically to core's rather than in a second, divergent system.*
- The CLI scaffolds modules with the abstraction already wired, so the default output is compliant.
- The library itself is an implementation choice. A well-maintained, widely-used i18n package is the
  expectation; naming one here would be the requirements doc legislating tooling.
- *Fallback behaviour is the one thing worth specifying: a missing translation falls back to the
  default language, and a missing key falls back to the key itself rather than rendering empty —
  visible in the UI as a detectable defect rather than a silent blank.*

**2. Image processing — `sharp`, queued through the job system. DECIDED.**
- Transformations are a **core capability modules consume**, consistent with media and taxonomy
  ([L149](overview.md#L149)), behind a storage-agnostic interface so it works identically on local
  filesystem and S3-compatible backends ([L153](overview.md#L153)).
- **`sharp`** is the choice: fast, maintained, native, and the de facto Node choice. ImageMagick is
  more powerful but a significantly heavier install, and its power is not needed for resize/reformat.
- **Transformations run as queued jobs through [O-6](open-gaps.md#L609), never inline in the request.**
  *This matters for both correctness and resilience: an on-demand transform of a large upload would
  hold an HTTP request open, and under [O-11](open-gaps.md#L930) an unbounded request is exactly the
  failure mode that exhausts the platform. Queuing also makes a slow transform visible and retryable
  instead of invisible.*
- **Originals are immutable; transformations are derived and regenerable.** A stored original plus a
  transformation recipe means a changed recipe re-renders rather than requiring re-upload — and it
  keeps the storage layer honest about which bytes are the source of truth.
- **Untrusted input is the security requirement**, not transform options: file type and declared
  content must be validated on upload, and the transform pipeline must never execute anything
  derived from the file. *Distinct from, and additional to, [O-16.5](open-gaps.md#L1578) — that one
  sanitises rich text, this one handles binary input.*

**Already settled and needing nothing further: rich-text sanitization** — allowlist-based, on write and
on render, core-owned, agents on the identical path ([O-16.5](open-gaps.md#L1578)).

#### GraphQL — DROPPED

**Decision: REST is the only read/write API surface.** GraphQL is not part of the platform, and no
module or extension point may introduce a second query surface.

**Why — against the standard case for GraphQL.** GraphQL's genuine advantages are client-composed
queries for an unknown client, and avoiding over/under-fetching. Neither applies:

1. **Vaah controls its dashboard.** The main consumer is a first-party React SPA whose developer also
   owns the backend, so purpose-built endpoints are cheap — and already produced by CRUD scaffolding
   ([overview.md L100-119](overview.md)). Over-fetching has a REST-native answer in what was already
   chosen: JSON:API conventions with `relationships` and `included`.
2. **The agent surface is MCP, not GraphQL.** Agents use the MCP tool contract — discrete, described,
   permissioned, audited operations ([overview.md L277](overview.md#L277)). That suits an agent better
   than a type graph, which is hard to reason about and harder to express field permissions over.
   Handing an agent a GraphQL endpoint would be a step backwards from what was already designed.
3. **The one genuinely GraphQL-shaped need — dynamic composition for the Reporting agent — is already
   solved** by the reporting query builder (see Reports above), which is purpose-built over the
   service layer and needs no general query language underneath.

**The decisive argument is field-level permissions.** [L55](overview.md#L55) requires field-level
read/write enforced at code level before operations. In REST that is per-endpoint and tractable. In
GraphQL it becomes per-field across a type graph — the classic GraphQL authorization hole. **GraphQL
would have made the platform's hardest security requirement harder.**

Secondary costs avoided: GraphQL is typically POST, forfeiting `ETag`/`Cache-Control`/304 caching;
partial failure returns HTTP 200 with errors alongside partial data, requiring design work at every
call site; and one endpoint containing N operations is harder to observe and rate-limit per resource.

**Cost accepted:** if third-party developers later want to build custom dashboard widgets with
unpredictable data needs, ad-hoc endpoints will be added then. That is incremental and much cheaper
than carrying two read surfaces with two authorization paths from v1.

**All references updated** in overview.md: the extension-point list (L90), CRUD scaffolding (L107),
agent-native discovery (L277), the API conventions preamble (L327), the conventions list (L339 — now
a positive "no second query surface" requirement rather than a deletion), cross-surface consistency
(L350), and the glossary (L509).

---

### O-16 · Cross-cutting gaps found in review — **ALL DECIDED**

Five gaps surfaced during a full read-through of [overview.md](overview.md) rather than from the
original list. Each was found by grepping the document for the mechanisms it *depends on* and finding
nothing. **Two of these are silent data loss**, created by decisions already taken.

---

#### 16.1 · Concurrent edits have no conflict rule 🔴

**The gap.** [L120](overview.md#L120) promises "an edit by one user reflects immediately to all
users" — real-time sync between users. Nothing anywhere in the document says what happens when **two
people edit the same record**. Grepping for optimistic locking, version conflicts, and lost updates
returns nothing. The implied behaviour is last-write-wins, which means **an agent silently overwrites
a human's in-progress edit** — and bulk agent actions ([L125](overview.md#L125)) operate on records a
human may be editing at that moment.

**Decision — optimistic locking, and agents never silently win:**

- Every mutable record carries a **version column**. An update states the version it read; if it
  changed underneath, the update is **refused with a conflict**, not applied.
- The UI shows **both versions** — the user's and the current one — and asks which to keep, rather
  than choosing silently on the user's behalf.
- **Agent writes obey the same rule.** An agent that reads a record and later writes a stale version
  receives a conflict, not a clobber; the agent loop re-reads and retries. This makes agent behaviour
  *honest* rather than quietly destructive, which matters more here than elsewhere because agents
  run unattended.
- **Bulk agent operations take a row lock or declare themselves destructive.** "Disable 214 users"
  cannot meaningfully conflict per-row, so a bulk action either runs in a single transaction or is
  explicitly marked as overwriting — and that marking appears in the audit entry specified at
  [L125](overview.md#L125).
- **Real-time delivery needs no conflict detection of its own** — SSE push is a *notification* that
  data changed; the version check happens at write time. This keeps the transport dumb and the
  correctness in one place.
- *Cost: one column per table, a check on every write, and a conflict-resolution UI. Small — and it
  is already implied by the real-time promise, because if two users see the same screen, one of them
  is currently going to be surprised.*

---

#### 16.2 · Soft delete contradicts data erasure 🔴

**The gap.** [L109](overview.md#L109) makes **soft delete the default** for every scaffolded entity —
`deleted_at`, recoverable. O-13 promises data deletion removes a person's data across core and module
data. A soft-deleted row **still contains the personal data**, so "data deleted" and "row soft-deleted
and still present" are the same state. The deletion workflow currently cannot do what it promises.

**Decision — two distinct operations, erasure hard-deletes and overrides recoverability:**

- **Business delete** — soft, recoverable, the platform default. Removing a draft article.
- **Privacy erasure** — hard, permanent, personal data gone, triggered by a data-deletion request.
- **When a privacy erasure targets a record, it hard-deletes regardless of the record's state** —
  active or soft-deleted. Recovery does not apply to erasure.
- **Recorded consequence, and it is a real edge case:** a soft-deleted record that is later erased
  cannot be restored, and a UI still offering "restore" on it will restore nothing. The requirements
  must state that restore is unavailable for erased records, so the affordance is hidden rather than
  failing at click time.
- *Cost: an escape hatch that is deliberately absent. This narrows a promise the document currently
  makes — note that "permanently delete" remains an ordinary UI action ([L110](overview.md#L110)); it
  simply is not what erasure triggers.*

---

#### 16.3 · The event bus is load-bearing and unspecified 🔴

**The gap.** [L95](overview.md#L95) says modules emit and subscribe on "the event bus that drives
workflow triggers and real-time updates." Three systems depend on one mechanism that appears
**exactly once in the whole document** — no semantics, no transport, not even a Tech Stack line.

**Decision — Redis-backed, following from D-4 and O-6 rather than introducing anything new:**

- **Transport: Redis pub/sub.** D-4 already put real-time on Redis and O-6 already requires it, so
  this consolidates onto one Redis dependency instead of adding a second. It also means an event can
  fire a workflow **after a restart**, which an in-process bus cannot do.
- **Two delivery tiers, because the two consumers need opposite things:**
  - **At-most-once for in-process subscribers** (UI live updates). A dropped refresh is
    self-correcting — the next event or poll fixes it, so retrying would be waste.
  - **Durable via the transactional outbox** for anything that must not be lost (a workflow trigger,
    an email). Written in the same transaction as the business change, then published — the O-6
    outbox doing double duty.
- **Events are namespaced and typed**, so a module's events cannot collide with core's and
  subscribers can filter.
- **Module subscriptions are declarative** — a module declares "I subscribe to `order.placed`" — which
  makes its event surface visible in the UI rather than hidden in code, and makes it auditable.
- *The event catalogue is a documented extension point; modules already depend on it.*

---

#### 16.4 · Frontend module loading is undefined 🔴

**The gap.** Modules ship default UI (O-1, CLI-generated) and [L87](overview.md#L87) says they
contribute routes and pages "via the frontend module registry" — the registry is *named* and never
explained. The admin panel is a **React + Vite SPA** ([L506](overview.md#L506)), and separately-built
third-party React code must be loaded into it at runtime. Never decided.

**Decision — runtime-loaded, so installing a module does not require rebuilding Vaah:**

- **The argument is the marketplace.** If modules are bundled at build time, installing one means
  rebuilding and redeploying Vaah — which makes the marketplace ([L75](overview.md#L75)) pointless,
  because every install becomes an ops action. Runtime loading is the requirement that makes the
  distribution model work.
- Modules ship as **runtime-loadable bundles**, loaded on demand and registered in the Zustand store
  already designated for the module registry ([L506](overview.md#L506)).
- Modules **register routes, menu items, and components at runtime.** Activation is data, not a
  rebuild.
- **Shared dependencies are supplied by the host** — React, design tokens, the query client. A module
  must not ship its own copy. *This is the single most common failure mode in runtime-loaded module
  UIs (duplicate-React crashes) and is a hard rule, not advice.*
- **A module whose UI fails to load does not break the panel** — same failure-containment posture as
  O-4: skip, disable, log, flash.
- **Version compatibility is checked before load**, using the `vaah_compatibility` range from O-14.
- **Modules consume core design tokens** (O-2), so theme changes apply without a rebuild.
- **CSS scope: shared with the host for v1.** Sharing is simpler and consistent with HMVC; the risk is
  a module's styles leaking into the panel. Sandboxing is recorded as a later hardening step, not a
  v1 requirement. *Deliberate deferral: consistency of the assembled UI matters more than isolation
  guarantees in the first release.*
- *Cost: this is the substantial one of the five. Runtime module loading has real failure modes
  (shared-deps mismatch, style leakage). It is expensive precisely because it is the answer the
  marketplace requires.*

---

#### 16.5 · Rich-text sanitization is a security requirement 🟡

**The gap.** Carried from O-15's routine-dependency list, elevated because it is a **security
control** rather than a formatting convenience. Both **users and agents** produce content, and a
self-hosted deployment means one stored XSS has full access to that instance. Guardrails
([L297–300](overview.md#L297-L300)) cover prompt injection and output schema validation, but nothing
covers sanitizing content before it is stored and re-rendered.

**Decision — sanitize on store *and* on render, as a core service modules must use:**

- **Sanitize on write**, at the platform's rich-text service — not per module, and not only at
  display. Storing clean content makes every reader safe by default, including API consumers and
  exports.
- **Sanitize again on render** — defence in depth, since content also arrives via migration, import,
  and seeded fixtures.
- **Allowlist, never denylist.** Denylists fail on every novel payload; an allowlist of permitted tags
  and attributes fails closed.
- **Link and URL safety**: block `javascript:` and `data:` URIs, force `rel="noopener noreferrer"` on
  `target="_blank"`.
- **Agent-generated content uses the identical path as user content** — no privileged route. Agent
  output already passes schema validation ([L300](overview.md#L300)); sanitization is the additional
  step for content that becomes HTML. *An agent that can write unsanitised HTML is a privilege
  escalation of the sanitization control, which is exactly the pattern the agent security boundary
  ([L230](overview.md#L230)) exists to prevent.*
- **Core owns the service; modules consume it**, consistent with media and taxonomy being core
  capabilities. Modules writing raw HTML is the gap being closed.
- *Cost: one service, one allowlist, a maintained sanitizer dependency. Low effort, high value.*

---

### O-18 · Email deliverability scope — **DECIDED**

**The gap.** Notifications & Email specified sending and bounce/complaint *status* but never said
whose job it is to make mail land in the inbox. For a self-hosted platform sending from the
operator's own domain, the answer is not obvious and the default assumption (that the framework
handles DNS and reputation) is wrong.

**Decisions:**

1. **Not Vaah's responsibility: deliverability.** SPF, DKIM, DMARC, domain reputation, inbox
   placement. Under [D-7](open-gaps.md) the operator owns the sending domain and configures DNS at
   their registrar. **Vaah manages no DNS records.** *A deployment sending from an unconfigured
   domain landing in spam is the correct, predictable outcome — not a platform defect.*
2. **Vaah's responsibility: send correctly, and know what happened.** Per-message delivery status
   (sent / delivered / bounced / complained, with the provider's reason) in a delivery log, so the
   operator can answer "did this actually go out?" without leaving Vaah.
3. **A suppression list for bounced and complained addresses**, with sending skipped and logged
   rather than retried. *The reason is sender reputation, not just the dead mailbox — continuing to
   mail a dead address degrades delivery for everyone else on the deployment.* Reversible by the
   operator.
4. **Transient and permanent bounces are distinguished** — a 4xx retries within the job's declared
   policy ([O-11](open-gaps.md)); a 5xx does not retry, is recorded, and surfaces as a dead-letter.
   This is [O-11](open-gaps.md)'s "declared retry policy, not automatic" applied to email.
5. **Provider webhook plus polled fallback**, so bounce handling does not depend on the webhook
   being reachable.
6. **Unusable configuration is visible** — an unauthenticated sending domain or a failing SMTP
   connection surfaces as a health-check warning ([Observability](overview.md)), not a queue that
   silently backs up.

---

### O-19 · Module activation locking — **DECIDED: no new requirement**

**The gap raised.** Activation runs migrations with platform credentials against the *running*
application. [O-11](open-gaps.md) requires distributed locking for scheduled jobs and single-flight
work, so the absence of a locking requirement for activation looked inconsistent — two operators
activating conflicting module versions, or an activation racing a live request.

**Decision: the existing locking coverage is sufficient. No new requirement.**

- **[O-11](open-gaps.md)'s distributed locking is a general platform guarantee, not a job-specific
  one.** Activation is single-flight work by definition, so it falls under the existing rule rather
  than needing a module-specific addition.
- **The realistic collision is operator error, not concurrency.** Two operators activating conflicting
  versions of the same module is prevented by the **sanity checks** at activation — a conflicting
  dependency or version is *refused before any migration runs* (see [Modules](overview.md)). The
  check is what prevents the damage; a lock would only serialise two operations that should not both
  be permitted anyway.
- **Migrations are guarded by backup + dry-run** ([Disaster Recovery](overview.md)), and expand/contract
  makes each individual migration step safe to have in flight. *A lock narrows the window; it does not
  make the operation safe. Backup, dry-run, and expand/contract are what make it safe.*
- **The residual race is a single operator activating during traffic.** That is inherent to running
  schema changes against a live application, and expand/contract is the designed mitigation —
  deliberately the same answer as
  [O-3](open-gaps.md)'s refusal to treat it as a solvable platform problem.

*Recorded so the question is not re-raised as a fresh gap.*

---

### O-17 · How modules extend the admin panel itself — **DECIDED**

**The gap.** [Extension Points](overview.md) settled what a module can *add*: menu items, routes,
dashboard widgets, settings sections. Every one of those creates a new surface. Nothing settled what a
module can do **inside an existing Vaah screen** — a Blog module's posts on a User detail page, its
column in the activity table, its button on a record page. Without a rule here the assembled panel has
no defined shape, which is the actual meaning of "extremely high extendability".

**Decisions:**

1. **Named, core-declared UI extension slots — not free injection.** A module fills a slot; only a
   core change creates one. This is the load-bearing choice: it is what lets Vaah promise a stable
   assembled UI while still letting modules reach into core screens. Free injection would mean any
   module can disfigure the core layout, at which point Vaah can promise nothing about how the panel
   looks — the same reasoning as [O-4](open-gaps.md): Vaah buys uniform governance for modules by
   giving up isolation from them, and this is the same trade on the UI side.
2. **Slot catalogue — two categories.**
   - *Adding a surface* (already decided): dashboard widget, left-menu item, top-menu item,
     profile-dropdown item, Settings section.
   - *Extending a core screen* (decided here, all four): **record-detail tab**, **list-view column**,
     **record-page action button**, **related-items panel**.
3. **Slot fills are declarative**, carrying target screen, slot, label, a **render condition**, and a
   **sort weight**. Core evaluates conditions — a module does not get to decide whether it belongs on
   a given record.
4. **A missing slot degrades that fill only.** A module built against a newer core targets a slot that
   does not exist in the running version: that fill is skipped with a logged warning and the module's
   own pages still load. This is the concrete payoff of `vaah_compatibility` being a *range*
   ([O-14](open-gaps.md)) rather than a pin — the range now means something specific instead of being
   a bare version claim.
5. **Ordering is module-declared weight**, tie-broken by module name. Chosen over alphabetical
   (no author control — "Support" lands wherever the alphabet puts it) and over install order (the
   menu would then differ between two deployments holding the same module set, which is exactly the
   kind of thing that wastes an afternoon of debugging).
6. **Slot content is permission-gated and attributed like any other UI**, and inherits the platform's
   focus/keyboard conventions rather than reimplementing them ([O-12](open-gaps.md)).

**Declared consequence, not an additional grant:** slot content runs inside the admin panel's origin,
so it holds the same power as the logged-in user. Stated here so it is a choice the design makes
rather than one it inherits from [O-4](open-gaps.md).

---

## Appendix A — Real-time debate: SSE vs WebSocket *(for D-4)*

**Recommendation: SSE for server→client push, ordinary REST POST for client→server.**

**Why this use case.** All four real-time needs are one-directional: record-change fan-out, live
dashboard widgets, in-app notifications, and agent chat progress. Nothing needs a bidirectional
channel. (`EventSource` is GET-only — client→server is a normal POST.)

**Three things decide it:**

1. **Auth gets simpler, not harder.** `EventSource` sends cookies, so an HttpOnly session cookie
   works with zero new code and the token never appears in a URL, referrer, or access log.
   WebSocket *also* cannot set an `Authorization` header — so it buys no security advantage and adds
   a guard surface.
2. **TanStack Query aligns with at-most-once delivery.** `invalidateQueries` is idempotent, so a
   dropped message degrades to "one missed refetch" that self-heals on focus/interval refetch. A
   WebSocket layer means building reconnection, ordering, and replay logic to reach the same
   idempotent call.
3. **Cheaper infra.** socket.io across instances additionally needs either `transports:
   ['websocket']` or sticky sessions. SSE is ordinary HTTP and passes through proxies that block the
   `Upgrade` handshake.

**Required mitigations — non-negotiable:**

| Item | Value |
|---|---|
| Heartbeat | SSE comment `:hb\n\n` every **15s** (under Cloudflare's 125s origin read timeout, which is **not** tunable below Enterprise) |
| Compression | **Must** exclude `text/event-stream` from app compression or the stream is silently buffered |
| nginx | `proxy_buffering off`, `proxy_read_timeout 3600s`, `proxy_cache off`, `proxy_http_version 1.1`, `proxy_set_header Connection ""` |
| Response headers | `Cache-Control: no-cache, no-transform`, `X-Accel-Buffering: no`, `Connection: keep-alive` |
| Backpressure | Coalesce high-frequency streams to full-current-state at ~10–20 events/sec; drop intermediates |
| Ordering | Per-tenant `seq` in every payload; client drops `seq <= lastApplied` |
| Auth placement | Resolve tenant + auth in a **guard**, not the handler — `@Sse()` flushes response headers *before* the handler runs, so a handler-thrown auth error yields an open stream, not a 401 |
| Client | **One refcounted stream per tab**, not one per component — a naive `useEffect` per component blows past browser connection limits and double-fires under React StrictMode |
| Internal transport | Redis pub/sub is **at-most-once by design** — treat it as a lossy signal and keep correctness independent of it |

**Would flip the recommendation:** collaborative editing/presence, high-frequency client→server push
(cursors, typing), or a need for **durable lossless ordered delivery** (audit/event-sourcing). For
that last case neither protocol is the answer — keep the SSE surface, move the transport behind it to
a transactional outbox or a real broker.

**Explicitly not a factor:** scale. At thousands of concurrent connections SSE is *more* efficient
than WebSocket (no upgrade handshake, no per-message framing). The scaling risk is on the Redis and
Node side, not the protocol.

## Appendix B — Search debate: Meilisearch vs Typesense *(for D-5)*

**Recommendation: PostgreSQL FTS for v1, behind a `SearchProvider` interface. Typesense later if
triggers fire. Meilisearch is the weaker choice of the two.**

**Why not a dedicated engine in v1:**

1. **Tenant isolation would get *worse*, not better.** Vaah already has a database per tenant — the
   strictest isolation available. A shared index with a `tenant_id` filter is *application*-enforced
   isolation; a bug in filter injection becomes a cross-tenant data leak. That is a security
   regression.
2. **Index count explodes.** Meilisearch's own docs cap around **~20 indexes** before performance
   degrades and warn that "accessing hundreds of databases rapidly degrades performance." The
   natural keyspace is tenant × entity — 20 tenants × 15 entities = 300. The module marketplace makes
   it unbounded.
3. **RAM is a self-hoster footgun.** Meilisearch's own measured example is **9.1 MB raw → ~305 MB
   RAM** (~30× expansion) and it auto-caps indexing at two-thirds of available RAM. On a 2 GB VPS
   already running Postgres + NestJS + Redis, it will fight Postgres for memory. Typesense publishes
   a sizing formula (~20 MB base + 2–3× indexed-field size) and lands several hundred MB at the same
   scale.

**Governance — the tiebreaker.** Meilisearch moved to a **dual-license model in August 2025**; core
search is still MIT but sharding went to BSL. Typesense is **GPL-3.0, bootstrapped, no VC ever** —
no investor board can force an open-core pivot. For an open-source core that ships to self-hosters
and plans a marketplace, governance direction is a first-class concern. (Note: third-party license
trackers still list Meilisearch as pure MIT — that is stale.)

**Correction to a common claim:** Typesense did historically require explicit `facet` fields, but
Meilisearch does **not** get a pass here either — both require pre-declaration
(`filterableAttributes` vs `facet: true`). For module-generated dynamic entities it's a wash, since
we generate the schema either way.

**Where Meilisearch is genuinely better:** async index deletion + an **atomic `swap-indexes`**
primitive (clean zero-downtime reindexing when a module changes an entity's searchable fields), a
mature zero-dependency JS client, and `disableOnAttributes` (turn typo tolerance off for ID/UUID
columns on dynamic entities).

**Migration triggers — move to a dedicated engine when any is true:**
- Per-tenant corpus > ~1M rows
- Tenants > 50 **and** entities > 25
- Typo tolerance beyond what `pg_trgm` covers (phonetic matches, 3–4 char typos, CJK)
- Sub-20ms p95 on complex faceted queries
- Unified relevance ranking across entities in global search
- Search becomes a headline feature rather than table search

**Interim step 1.5 worth knowing about:** a Postgres extension (BM25 ranking + built-in typo
tolerance, in-database) closes the main Postgres FTS gaps — no new service, isolation story intact.
Evaluate that before considering anything external.

**The thing actually worth protecting is the `SearchProvider` abstraction**, not any particular
backend. Migration cost stays low because of it.

## Appendix C — Verified version baseline (2026-10-11)

Reference only — do **not** copy these into `overview.md`. Pin in `STACK.md` + lockfiles.

| Package | Version | Note |
|---|---|---|
| `@nestjs/core` / `common` | 12.1.2 | ESM-first; v11 on `legacy` tag |
| `@nestjs/swagger` | 12.0.2 | emits OpenAPI **3.0.0** by default (verified in source) |
| `@modelcontextprotocol/server` | 2.3.1 | v2 scoped packages; the current stable line |
| `@modelcontextprotocol/sdk` | 1.32.1 | legacy — support window ends ~Jan 2027 |
| `@a2a-js/sdk` | 1.3.0 | official A2A SDK (unscoped `a2a-js` on npm is a different, unofficial package) |
| `drizzle-orm` | 0.45.4 | thin driver — cheap per-tenant clients |
| `@nestjs/drizzle` | 0.0.1 | first-party, brand new — **0.x** |
| `@prisma/client` | 7.10.0 | pin this; the `prisma` CLI `latest` tag is an 8.0 rc |
| `bullmq` | 6.3.12 | v6 removed `repeat` from `Queue.add()` → `upsertJobScheduler()` |
| `zod` | 4.6.5 | Standard Schema; single schema language across HTTP/AI/MCP |
| `vitest` | 5.0.3 | NestJS 12's default test runner |
| `react` / `vite` | 19.3 / 8.x | Vite not displaced; React Compiler v1 stable |
| `tailwindcss` | 4.x | `@theme` cannot be scoped — see O-2 |

⚠️ Several first-party NestJS reliability packages (outbox, workflows, locks, idempotency,
webhooks) are **all pre-1.0 and weeks old**. The design is right; the APIs are a bet. Pin exact
versions and re-evaluate quarterly.