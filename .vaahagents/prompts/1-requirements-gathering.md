---
description: Orchestrate multi-agent requirements structuring for VaahApp — audit the existing requirements corpus, resolve open gaps and doc drift, and generate a structured, dependency-ordered feature set with a manifest for the milestones-and-tasks pipeline
version: 1.0
auto_execution_mode: 3
---

# Vaah Requirements Structuring

## Purpose

Orchestrate a multi-agent workflow that transforms Vaah's existing requirements corpus into validated, dependency-ordered software requirements with structured documentation aligned to IEEE 29148, ISO 25010, and agile RE best practices. Output both human-readable markdown and a machine-readable manifest consumed by the downstream pipeline (milestones → tasks → execution → tests).

**Corpus-first, not greenfield.** Vaah's requirements already exist as a curated corpus:

- `overview.md` — the canonical requirements overview (the single source of truth for *what Vaah is*)
- `open-gaps.md` — decided stack choices (`D-*` items) and open/decided gaps (`O-*` items)

This prompt does **not** re-ask what is already decided. Its job is to (1) audit the corpus, (2) resolve remaining open gaps, contradictions, and doc drift, (3) convert the corpus into a structured, traceable, dependency-ordered feature set that the milestones-and-tasks pipeline can consume.

## Variables

- **`{{PROJECT_NAME}}`** *(static)* — `VaahApp` — the agent-native HMVC software development platform (open-source NestJS core + installable modules).
- **`{{PROJECT_ROOT}}`** *(static)* — `/Users/pk/Projects/vaah`
- **`{{REQUIREMENTS_DIR}}`** *(static)* — `{{PROJECT_ROOT}}/.vaahagents/requirements/`
- **`{{OVERVIEW_FILE}}`** *(static)* — `{{REQUIREMENTS_DIR}}/overview.md` — canonical requirements overview.
- **`{{GAPS_FILE}}`** *(static)* — `{{REQUIREMENTS_DIR}}/open-gaps.md` — decided stack + open gaps (`D-*` / `O-*` items).
- **`{{FEATURES_DIR}}`** *(static)* — `{{REQUIREMENTS_DIR}}/features/`
- **`{{MANIFEST_FILE}}`** *(static)* — `{{REQUIREMENTS_DIR}}/.requirements-manifest.json` — machine-readable manifest for downstream pipeline consumption.
- **`{{MILESTONES_DIR}}`** *(static)* — `{{PROJECT_ROOT}}/.vaahagents/milestones-and-tasks/` — downstream pipeline output.
- **`{{MAX_RESEARCH_DEPTH}}`** *(static)* — `3` — maximum rounds of iterative gap analysis before surfacing unresolved items to user.
- **`{{RISK_FOCUS_COUNT}}`** *(static)* — `5` — number of highest-uncertainty items to prioritize in each elicitation round.

## Corpus Map (navigation only — the files are authoritative)

`overview.md` H2 sections: Overview · Design Priorities · Features · UI & Design · Developer Experience · RBAC · Users · Modules · Licensing & Distribution · Extension Points / Module Extension Model · Module Architecture · CRUD Scaffolding · Audit Logs · Settings · Registrations · Media & Taxonomy · Data & Storage · Notifications & Email · Reports · Workflows / Automation · Onboarding · Agentic Platform · Agents · Agent Operations · Agent-Native Platform · Model Management & Guardrails · Agent Evaluation · Platform · Deployment model · Localization (i18n) · API Design Conventions · API Authentication · Data Privacy & Consent · Security & Data Protection · Observability · Performance · Deployment & Self-hosting · Configuration & Feature Flags · Disaster Recovery · Resilience & Failure Handling · Accessibility · Module trust & the marketplace · Event bus · Frontend module loading · UI Extension Slots · Content Sanitization · Delivery & Release Engineering · Developer documentation · Platform Standing & Adoption · Tech Stack · Terminologies.

`open-gaps.md` items: `D-1`…`D-8` (decided: OpenAPI 3.0.x · MCP SDK v2 · NestJS 12 · SSE · PostgreSQL FTS · admin-panel theming · self-hosted is total · single-tenant) and `O-1`…`O-19` plus sub-items (O-5b, O-15.x, O-16.1–16.5) — verify current status at runtime; statuses in this map drift.

## Instructions

- **Corpus-first rule:** `{{OVERVIEW_FILE}}` and `{{GAPS_FILE}}` are the authoritative requirements corpus. Never re-ask, re-research, or re-decide what is already recorded there. Elicitation targets ONLY items that are open, missing, or contradicted at runtime.
- **Decision provenance:** Every requirement ID generated here must trace to a `overview.md` section or a `D-*` / `O-*` item in `open-gaps.md`. No orphan requirements.
- **No version numbers in requirements docs:** Version pins live in `STACK.md` + lockfiles, never in `overview.md` or generated feature docs — a requirements doc that carries version numbers goes stale.
- **Phase-level flow control:** Each phase has a clear goal and completion gate. Only one phase `in_progress` at a time. Mark `completed` immediately after finishing — do not batch completions.
- **Place methodology decision tree** in the Workflow as Phase 1 Step 4 — DO NOT put it in Instructions.
- **Tool-first:** Use Read, Edit, Write, and Grep tools for all file operations — never Bash (`cat`, `find`, `ls`, `rg`).
- **Context compaction survival:** After Phase 2 and again after Phase 5, write an inline `### Checkpoint: status` summary listing: completed phases, generated files, decisions made, and next phase goal. This ensures state survives compaction.
- **Error recovery — MCP failures:** IF the `memory` MCP server is unreachable, THEN log the error, note "Memory update deferred" in the report, and continue. IF `context7` fails for a library lookup, THEN note the lookup and fall back to the corpus's own verified-version notes. Never block the entire workflow on a single MCP failure.
- **Error recovery — user contradictions:** IF the user's answers contain logical contradictions, THEN flag them in the analysis phase with a proposed resolution, rather than silently picking one side.
- **Error recovery — file write failures:** IF a Write tool call fails, THEN log the file path and error, retry once after 1 second, and IF still failing, mark the file as "write-failed" in the manifest and continue with remaining files.
- **Todo list:** Create a todo list at Phase 1 covering all 7 phases. Mark exactly ONE phase `in_progress` at a time. Mark `completed` immediately after finishing — do not batch completions.
- **Blast radius:** File writes (Write/Edit tools) are freely taken without confirmation — including the corpus sync that updates `overview.md` / `open-gaps.md` (Step 21). No git operations, publishes, or destructive commands occur in this workflow.
- **Sub-agent output:** When spawning sub-agents, each returns structured data (via schema). Collect all outputs before proceeding to the next phase. A sub-agent that returns null indicates user-skip or terminal failure — log it and proceed.
- **Pipeline contract:** This prompt generates `{{FEATURES_DIR}}/`, the core + conditional docs in `{{REQUIREMENTS_DIR}}/`, and `{{MANIFEST_FILE}}`. The downstream milestones-and-tasks pipeline depends on these outputs. Every feature ID must be traceable: `overview.md` section / `D-*` / `O-*` item → feature → acceptance criteria. The manifest is the machine-readable handoff.

## MCP Servers

| Server | Purpose | When to Use |
|--------|---------|-------------|
| `context7` | Authoritative library/framework docs to verify stack-dependent open items (NestJS 12, `@modelcontextprotocol/server` v2, MikroORM, BullMQ, Tailwind 4) | Phase 1 — Step 3: when an open item hinges on library behavior |
| `sequential-thinking` | Systematic gap analysis, contradiction detection, dependency graph validation | Phase 2 |
| `memory` | Cross-session persistence of project context and decisions | Phase 7 |
| Explore (agent) | Corpus audit — use ONLY if the corpus outgrows direct reading (> ~5000 lines combined) | Phase 1 — Step 1 |

## Skills

| Skill | When to Use |
|-------|-------------|
| `brainstorming` | Phase 1 — Step 3: surfacing non-obvious features and edge cases on open/missing items before presenting findings |
| `sequential-thinking` | Phase 2 — structured multi-step gap analysis, contradiction detection, and dependency-graph validation before presenting findings to the user |
| `prd` | Phase 5 — Step 19: when generating `prd.md` (executive summary, user stories, technical specification, risk analysis) |
| `architecture` | Phase 5 — Step 17: when generating `architecture.md` (evaluating the core/module architecture with trade-offs and consequences) |
| `architecture-patterns` | Phase 5 — Step 17: when modeling the core ↔ module boundary (bounded contexts, hexagonal/clean layering, avoiding dependency cycles) |
| `architecture-decision-records` | Phase 5 — Step 17: when writing the ADRs for key decisions in `architecture.md` |
| `security-review` | Phase 5 — Step 19: when generating `security/security.md` (RBAC + field-level permissions, OAuth 2.1, the MCP security boundary, prompt-injection defense, agent-as-actor attribution) |
| `ui-ux-pro-max` | Phase 5 — Step 19: when generating `ui/ui-design.md` (dense dashboard layout, dark/light mode, mobile responsiveness) |
| `ui-taste` | Phase 5 — Step 19: when generating `ui/ui-design.md` (anti-slop UI playbooks, visual review) |
| `frontend-design` | Phase 5 — Step 19: when generating `ui/ui-design.md` (high-quality frontend design) |
| `design-better` | Phase 5 — Step 19: when generating `ui/ui-design.md` (UX heuristics & craft catalogue — hierarchy, interaction, accessibility, motion, polish) |
| `shadcn` | Phase 5 — Step 19: when generating `ui/ui-design.md` — the frontend component library is shadcn/ui |
| `design-tokens` | Phase 5 — Step 19: when generating `ui/design-tokens.md` (light/dark palettes, spacing scale, type ramp, component-level tokens) |
| `design-system` | Phase 5 — Step 19: when generating `ui/design-tokens.md` (three-layer token architecture: primitive → semantic → component) |
| `tailwind-best-practices` | Phase 5 — Step 19: when generating `ui/design-tokens.md` (Tailwind 4.x `@theme`, admin-panel-configurable themes per D-6) |
| `agents-sdk` | Phase 5 — Step 19: when generating `agents/agent-platform.md` (server-side agent loop, MCP tools/resources/prompts, A2A surfaces, sub-agents) |
| `claude-api` | Phase 5 — Step 19: when generating model management & guardrails requirements (multi-provider, BYO keys, routing/fallback, pinning & recording) |

## Workflow

### Phase 1: Corpus Audit & Targeted Elicitation

**Goal:** Establish the true state of the corpus (decided / open / missing / stale), then elicit only what is actually unresolved.

**Step 1: Read the Corpus**

Read `{{OVERVIEW_FILE}}` and `{{GAPS_FILE}}` in full with the Read tool. Build a **state inventory** — one row per `overview.md` H2 section and per `D-*` / `O-*` item:

| Field | Values |
|-------|--------|
| `ref` | Section name or `D-*` / `O-*` ID |
| `status` | `decided` / `open` / `missing` / `stale` |
| `note` | One line — what is decided, what is unresolved, or what contradicts another item |

`stale` = a statement in `overview.md` that a `D-*` / `O-*` decision supersedes (doc drift — e.g. a section still saying "OpenAPI 3.1" after D-1 decided 3.0.x, or multi-tenancy language after D-8 decided single-tenant).

**Gate:** IF either corpus file is missing or empty, THEN stop and ask the user to restore it before proceeding.

**Step 2: Confirm Inventory & Scope**

Present the state inventory to the user, grouped:

1. **Decided** — count + any the user should double-check
2. **Open** — each item with its open question
3. **Stale (doc drift)** — each superseded statement with the decision that supersedes it
4. **Missing** — areas the corpus never covers that a platform of this scope implies

Then ask the scope question: **full structuring** (entire corpus → dependency-ordered feature set + all docs) or **gap resolution only** (resolve open/stale items, sync the corpus, regenerate the manifest — no feature docs). WAIT for the user's answer. Store as `on-the-fly:scope`.

**Step 3: Risk-Annealed Elicitation of Open Items**

IF the inventory has open or missing items: present a batch of follow-up questions ordered by **uncertainty × impact** — the most architecturally consequential unknowns first, `{{RISK_FOCUS_COUNT}}` per round.

- Invoke the `brainstorming` skill before presenting findings, to surface non-obvious features and edge cases the corpus never considered.
- IF an open item hinges on library behavior (e.g. an MCP spec question, a NestJS 12 ESM/HMVC question, a MikroORM migration question), THEN verify against `context7` before asking the user — present the verified fact, then the decision.
- Batch all questions in a round together. WAIT once per round. Store as `on-the-fly:gap-answers`.

IF the inventory has no open items, THEN skip to Step 4 and note "corpus fully decided".

**Step 4: Select Requirements Methodology**

Based on the project's complexity and domain, pick the most appropriate elicitation technique and integrate it into the remaining steps:

| If Project Is | Use This Technique | Why |
|--------------|-------------------|-----|
| **Simple CRUD / MVP** | User Story Mapping + MoSCoW | Lightweight, visual, gets to MVP fast |
| **Complex domain (finance, healthcare, logistics)** | Event Storming + Bounded Contexts | Surfaces domain events and aggregate boundaries |
| **UX-heavy (consumer app, dashboard)** | User Story Mapping + Example Mapping | Captures user journey with concrete examples |
| **Integration-heavy (API, platform, middleware)** | Specification by Example + Interface Analysis | Concrete contracts drive specs AND tests |

Vaah is a complex, integration-heavy platform (module extension points, MCP/A2A agent surfaces, event bus, third-party module trust) — expect **Event Storming + Bounded Contexts**, combined with **Specification by Example + Interface Analysis** for every external contract (REST/OpenAPI, MCP, A2A, module extension points).

**Gate:** IF the scope is *gap resolution only*, THEN note "methodology applied to open items only" and proceed.

**Step 5: Methodology Injection**

Apply the selected methodology to refine requirement depth:
- **Event Storming:** Identify domain events, commands, aggregates, bounded contexts across the module/core boundary. Store as `on-the-fly:domain-model`
- **Specification by Example:** For each external contract (REST, MCP, A2A, extension point), capture a concrete input-output example pair. Store as `on-the-fly:spec-examples`
- **Example Mapping:** For each ambiguous rule, ask for 1-2 concrete examples. Store as `on-the-fly:examples`

WAIT for user responses. If methodology produces new questions, ask them now.

**Step 6: Complete Elicitation & Confirm**

Present a consolidated summary of everything: confirmed decisions, resolved open items, stale statements to fix, methodology output, and quality attribute coverage (per ISO 25010 — see Phase 2).

**Gate:** WAIT for explicit user confirmation or corrections before proceeding to Phase 2.


### Phase 2: Analysis & Gap Detection

**Goal:** Run multi-perspective adversarial analysis on the full requirements (corpus + elicitation output) before generating documentation.

**Step 7: Quality Attribute Workshop**

Systematically verify non-functional requirements against ISO 25010. The corpus already covers most categories — verify and fill, do not re-ask:

| Category | Corpus Source | Verify / Elicit |
|----------|---------------|-----------------|
| **Performance Efficiency** | `Performance` section | Confirm concrete targets are still current |
| **Reliability** | `Resilience & Failure Handling`, `Disaster Recovery` | RPO/RTO, backup/restore, HA |
| **Security** | `Security & Data Protection`, `API Authentication`, `RBAC` | Compliance scope, data sensitivity |
| **Maintainability** | `Developer Experience`, `Module Architecture` | Team size, ownership model, lifespan |
| **Compatibility** | `Extension Points`, `API Design Conventions` | Interoperability targets |
| **Portability** | `Deployment & Self-hosting` | Self-hosting scope (D-7: total) |
| **Accessibility** | `Accessibility` | WCAG 2.2 AA — blocking CI gate |

Store responses as `on-the-fly:nfr-requirements`.

**Step 8: Adversarial Gap Analysis (3-Panel Review)**

Invoke the `sequential-thinking` skill to structure the multi-step analysis (gap enumeration → contradiction detection → dependency-graph validation) before spawning the panel.

Spawn THREE sub-agents in parallel, each with a distinct adversarial lens. Each agent independently reviews ALL collected data (corpus inventory, gap answers, methodology output, NFRs).

**Agent 8a — Completeness Lens:**
```
Identify missing, vague, or undefined elements in these requirements.
Focus on: undefined terms, vague scope boundaries, missing error handling,
absent edge cases, implicit assumptions. Pay special attention to the
module/core boundary: what does the core guarantee vs. what is each
module's responsibility? Return structured gaps.
```
Label: `gap:completeness` — Schema: `{"type":"array","items":{"type":"object","properties":{"severity":{"type":"string","enum":["critical","important","minor"]},"category":{"type":"string"},"description":{"type":"string"},"impact":{"type":"string"},"recommended_default":{"type":"string"}}}}`

**Agent 8b — Contradiction & Drift Lens:**
```
Find all logical contradictions, conflicting constraints, and inconsistent
assumptions across the gathered requirements. ADDITIONALLY cross-check
overview.md against open-gaps.md for doc drift: any overview.md statement
that a D-*/O-* decision supersedes (wrong version, superseded scope,
stale framing). Return structured contradictions and drift.
```
Label: `gap:contradiction` — Schema: `{"type":"array","items":{"type":"object","properties":{"severity":{"type":"string","enum":["critical","important","minor"]},"statement_a":{"type":"string"},"statement_b":{"type":"string"},"resolution":{"type":"string"},"drift":{"type":"boolean"}}}}`

**Agent 8c — Risk Lens:**
```
Identify high-risk assumptions, cost/effort underestimate signals, technical
debt traps, and dependency risks. Flag pre-1.0 / first-party dependencies
whose APIs are a bet, and any requirement that depends on uneven external
client support (e.g. ui:// MCP Apps). Return structured risks.
```
Label: `gap:risk` — Schema: `{"type":"array","items":{"type":"object","properties":{"severity":{"type":"string","enum":["critical","important","minor"]},"risk_type":{"type":"string"},"description":{"type":"string"},"mitigation":{"type":"string"}}}}`

**Gate:** Collect all three gap reports. Merge by severity. Only gaps where ≥2 agents agree (majority verdict) proceed as confirmed gaps. Single-agent findings are flagged as "possible — verify with user."

**Step 9: Loop-Until-Dry (Gap Resolution)**

Present confirmed gaps to the user in priority order. Use `{{MAX_RESEARCH_DEPTH}}` rounds maximum.

**Round N:**
1. Present Critical gaps first (must resolve now)
2. Present Important gaps (should resolve now)
3. For Minor gaps, offer recommended defaults that user can accept or override
4. WAIT for user responses
5. IF user resolves some gaps, re-run gap analysis on new information
6. IF no new gaps found or `{{MAX_RESEARCH_DEPTH}}` rounds reached, exit loop

**Step 10: Consolidated Requirements Confirmation**

Present the complete, consolidated requirements for approval:

- **Project Overview:** VaahApp — agent-native HMVC platform; objective, audiences (platform developers + LLMs), scale (single-tenant, self-hosted)
- **Tech Stack:** Decided stack per `open-gaps.md` (no version numbers)
- **Quality Attributes:** ISO 25010 coverage checklist with targets
- **Feature Landscape (Methodology-Enhanced):** Features derived from the selected methodology with development phases
- **Constraints & Decisions:** Hard constraints, trade-offs accepted, risky assumptions with mitigations
- **NFR Commitments:** Performance, reliability, security, and other quality targets
- **Resolved Gaps & Drift:** Summary of what was found and resolved

**Gate:** WAIT for user approval. IF changes requested, THEN incorporate and re-present. Do NOT proceed until user confirms.

**Checkpoint:** Write an inline `### Checkpoint: Phase 2 complete` summary. This survives context compaction.


### Phase 3: Dependency Planning

**Goal:** Produce a dependency-ordered feature list before generating any documentation.

**Step 11: Identify & Order Features by Dependencies**

Use the corpus sections + methodology output to identify all features. Apply **strict dependency ordering** — NOT priority ordering.

- Extract feature candidates from `overview.md` sections and domain events / spec examples
- For each feature, identify its dependencies (what must exist before it can be built)
- **Independent features first:** features with zero dependencies are identified and numbered first (F1, F2, ...) — expect the platform foundation (module system, event bus, RBAC core, CRUD scaffolding) to anchor the front. Only AFTER all independent features are placed do dependent features get numbered, each depending exclusively on already-numbered features
- Feature N may ONLY depend on features 1 through N-1
- **Every feature file must state its dependencies explicitly** — a `Dependencies` field in the metadata header (feature IDs or `none`) plus a one-line rationale for each dependency. A feature with no dependencies is marked `Dependencies: none (independent)`
- Validate the dependency graph: check for circular dependencies

**Step 12: Assign Attributes Per Feature**

For each feature in dependency order, assign:

| Attribute | Values |
|-----------|--------|
| **ID** | F1, F2, F3, ... (development order — independent features first) |
| **Phase** | MVP / Enhancement / Scale / Growth |
| **Priority** | Critical / High / Medium / Low |
| **Effort** | Small / Medium / Large / XL |
| **Dependencies** | Feature IDs this feature needs, or `none (independent)` — must all be lower-numbered features |
| **Source** | `overview.md#<section>` and/or `open-gaps.md#<D-*/O-* item>` — the provenance ref |
| **Acceptance Criteria Format** | Gherkin (`Given/When/Then`) or free-text checklist (per methodology selection) |

**Step 13: Present Dependency Map**

- Structured list of all features with IDs, phase, priority, source, and dependencies
- Text-based dependency diagram
- Sequential development rationale explanation
- **Gate:** WAIT for user confirmation of the feature order


### Phase 4: Validation (Specification by Example)

**Step 14: Concrete Examples Per Feature**

For each feature, capture 1-2 concrete examples following the methodology selected in Phase 1:
- **Event Storming:** Ask for specific event triggers and output events
- **Specification by Example:** Ask for input-output pairs (for external contracts: a concrete REST request/response, an MCP tool call/result, an A2A task exchange)
- **Example Mapping:** Ask for example rules and counter-examples

Store as `on-the-fly:feature-examples`.

**Step 15: Translate Examples to Acceptance Criteria**

For each feature, write acceptance criteria in the format appropriate to the methodology:
- **Event Storming:** Event → Command → Event flow
- **SBE:** Input → Action → Expected output table
- **Standard:** Structured checklist with testability markers

**Gate:** IF any acceptance criterion is ambiguous or untestable, THEN flag it for user clarification. Do not proceed with vague criteria.


### Phase 5: Specification Generation (Pipeline)

**Goal:** Generate all documentation files using independent sub-agents per feature. Run in dependency order — feature files for dependencies complete before dependent features start.

**Add checkpoint here for compaction survival:**
Write `### Checkpoint: Phase 5 generation starting` inline listing: total features, methodology, manifest target path, and docs directory. This survives context compaction during the long generation loop.

**Step 16: Generate Features (Independent First, Then Dependent — Pipeline)**

Generate in strict dependency order: **all independent features (zero dependencies) first**, then dependent features — a feature is generated only after every feature it depends on has passed verification. Run features through a pipeline. Each feature passes through two stages independently — NO barrier between features:

```
Feature F1 → [Writer Agent → Verifier Agent] → F1 passes to F2's verifier
Feature F2 → [Writer Agent → Verifier Agent] → F2 verifier sees F1's output
Feature F3 → [Writer Agent → Verifier Agent] → ...
```

**Stage 1 — Feature Writer Agent:**
```
Write a feature document at {{FEATURES_DIR}}/feature-NNN-slug.md with:
- Metadata header (feature ID, name, phase, priority, effort, source ref into
  overview.md / open-gaps.md)
- **Dependencies section** — explicit list of feature IDs this feature depends
  on (with a one-line rationale each), or `none (independent)`. Only
  lower-numbered features may be listed.
- Requirement statement traced to its source section
- BDD acceptance criteria
Feature ID: {{FID}}
Label: `write:feature-{{FID}}`
```

**Stage 2 — Feature Verifier Agent:**
```
Verify that {{FEATURES_DIR}}/feature-NNN-slug.md:
1. Has the correct metadata header
2. Acceptance criteria are testable (each criterion can be objectively verified)
3. All dependencies are correctly referenced
4. Source ref points to a real section/item in overview.md or open-gaps.md
5. Cross-references to other docs use relative links
6. Contains no placeholder text, TODO markers, or version numbers
Return structured verification: {passed: bool, issues: [string]}
```
Label: `verify:feature-{{FID}}` — Schema: `{"type":"object","properties":{"passed":{"type":"boolean"},"issues":{"type":"array","items":{"type":"string"}}}}`

IF verifier returns `passed: false`, THEN regenerate the feature file with specific fixes.

**Step 17: Generate Architecture Document**

Invoke `architecture` + `architecture-patterns` + `architecture-decision-records`, then write `{{REQUIREMENTS_DIR}}/architecture.md`:
- HMVC system components and interactions (core ↔ module boundary, text-based diagram)
- Decided tech stack with justification for each choice (cite the `D-*` / `O-*` item, no version numbers)
- Module extension points and the module lifecycle; repo topology (O-3)
- Event bus (Redis pub/sub, transactional outbox) and queue (BullMQ) topology
- Agent surfaces: Vaah MCP server, GenicUI, A2A Agent Card, spec-driven build-time agents
- Directory structure, environment configuration (dev, staging, production)
- Architecture Decision Records (ADRs) for key decisions

**Step 18: Generate Roadmap**

Write `{{REQUIREMENTS_DIR}}/roadmap.md`:
- Milestone 1: MVP — core platform foundation (dependency-ordered)
- Milestone 2: Enhancement
- Milestone 3: Scale
- Milestone 4: Growth
- Relative timeline, dependencies between milestones, risk per milestone
- Definition of done per milestone

**Step 19: Generate Conditional Documents**

Generate based on the corpus and user's answers. Run in parallel — they don't depend on each other. Invoke the matching skill from the Skills table where one applies:

- `{{REQUIREMENTS_DIR}}/prd.md` — invoke `prd` (executive summary, user stories, technical specification, risk analysis)
- `{{REQUIREMENTS_DIR}}/deployment/deployment.md` (self-hosted, total — D-7)
- `{{REQUIREMENTS_DIR}}/testing/testing-strategy.md` (core gates per O-7; module gates not Vaah's responsibility)
- `{{REQUIREMENTS_DIR}}/glossary.md` (from `overview.md` → Terminologies)
- `{{REQUIREMENTS_DIR}}/security/security.md` — invoke `security-review`
- `{{REQUIREMENTS_DIR}}/api/api-design.md` (REST canonical surface, OpenAPI 3.0.x, cursor pagination, agent-friendliness)
- `{{REQUIREMENTS_DIR}}/data-models/data-models.md` (MikroORM entities, field-level permissions, dual attribution columns)
- `{{REQUIREMENTS_DIR}}/ui/ui-design.md` — invoke `ui-ux-pro-max` + `ui-taste` + `frontend-design` + `design-better` + `shadcn` (dense dashboard, dark/light, admin-panel theming per D-6, UI extension slots, component library)
- `{{REQUIREMENTS_DIR}}/ui/design-tokens.md` — invoke `design-tokens` + `design-system` + `tailwind-best-practices` (semantic tokens, light/dark palettes, spacing/type scales, Tailwind 4.x `@theme`)
- `{{REQUIREMENTS_DIR}}/agents/agent-platform.md` — invoke `agents-sdk` + `claude-api` (MCP full surface per D-2, A2A, agents as first-class actors, model management & guardrails, agent evaluation)
- `{{REQUIREMENTS_DIR}}/integrations/integrations.md` (webhooks, outbound events, external MCP servers)
- `{{REQUIREMENTS_DIR}}/i18n/internationalization.md` (UI-string mechanism, UTC storage, RTL)
- `{{REQUIREMENTS_DIR}}/accessibility/accessibility.md` (WCAG 2.2 AA, blocking CI gate)
- `{{REQUIREMENTS_DIR}}/analytics/analytics.md` (Reports / BI per O-15)

**Step 20: Generate README**

Write `{{REQUIREMENTS_DIR}}/README.md`:
- Table of contents linking to all generated docs (core + conditional)
- Key features summary, tech stack summary
- Quick links to all documentation files
- List of conditional docs generated and why

**Step 21: Sync the Corpus**

Apply the confirmed resolutions back into the corpus — this is what `open-gaps.md` asks of every DECIDED item ("needs to be reflected in `overview.md`"):

1. In `{{GAPS_FILE}}`: mark each resolved open item `DECIDED` with a short decision record; record any newly decided stack choice as the next `D-*` item.
2. In `{{OVERVIEW_FILE}}`: fix every stale statement from the drift report (superseded versions, scope, framing) and add sections for any newly decided areas.
3. Preserve the `D-*` / `O-*` numbering scheme and the doc's existing voice and structure — surgical edits, not rewrites.
4. **Gate:** re-run the drift check on the two files; IF any superseded statement remains, THEN fix it before proceeding.

### Phase 6: Verification

**Step 22: Generate Requirements Manifest**

Write `{{MANIFEST_FILE}}` (JSON) containing:
```json
{
  "manifest_version": "1.0",
  "project": "VaahApp",
  "generated_at": "<ISO date>",
  "methodology": "<selected methodology>",
  "corpus": {
    "overview": "overview.md",
    "gaps": "open-gaps.md",
    "decided_items": ["D-1", "D-2", "..."],
    "sync_status": "complete | partial"
  },
  "tech_stack": { ... },
  "features": [
    {
      "id": "F1",
      "name": "...",
      "priority": "Critical",
      "phase": "MVP",
      "effort": "Medium",
      "dependencies": [],
      "source": "overview.md#rbac",
      "file": "features/feature-001-slug.md",
      "acceptance_criteria_count": 5
    }
  ],
  "quality_attributes": {
    "performance": { "elicited": true, "targets": { ... } },
    "accessibility": { "elicited": true, "targets": { "wcag": "2.2 AA", "gate": "blocking CI" } },
    "...": {}
  },
  "files": ["README.md", "architecture.md", ...],
  "traceability": {
    "corpus_to_features": [ ... ],
    "features_to_acceptance_criteria": [ ... ]
  },
  "pipeline_handoff": {
    "next_prompt": "milestones-and-tasks generator",
    "output_dir": ".vaahagents/milestones-and-tasks/",
    "manifest_format_version": "1.0"
  }
}
```

**Step 23: Verification Ladder**

Run checks from fastest to slowest:

1. **File existence** — List `{{REQUIREMENTS_DIR}}` to confirm all expected files and folders exist
2. **Metadata check** — Read first 10 lines of each generated file to verify header and structure
3. **Content depth** — Spot-check 3 files for substantive content (not just stubs or ToC-only)
4. **Traceability check** — Confirm every feature ID in the manifest has a corresponding file with a valid source ref, and every file corresponds to a confirmed requirement
5. **Cross-reference validity** — Ensure all `[relative links](./...)` in generated files point to existing files
6. **Manifest consistency** — Feature count, file list, and IDs must match between manifest and filesystem
7. **Corpus sync check** — No open item in `open-gaps.md` still marked open without a recorded decision; no superseded statement remains in `overview.md`

**Gate:** IF any file is missing, malformed, or empty, THEN regenerate it. IF the manifest is inconsistent with the filesystem, THEN fix the discrepant entry. Do NOT proceed until verification passes.

**Checkpoint:** Write `### Checkpoint: Phase 6 complete` inline listing: total files, manifest path, verification results (pass/fail per file). This survives compaction.


### Phase 7: Memory & Pipeline Handoff

**Step 24: Update AI Memory**

Save to the `memory` MCP server:
- Entity: `VaahApp` (type: `project`)
  - Observations: project root, requirements dir, description, tech stack, methodology
- Entity: `VaahApp-requirements` (type: `requirements-manifest`)
  - Observations: manifest path, feature count, files generated, key decisions, NFR targets
- Entity: `Pipeline State` (type: `pipeline-state`)
  - Observations: `1-requirements-gathering.md → COMPLETE → ready for milestones-and-tasks pipeline`
  - Observations: `manifest_path: {{MANIFEST_FILE}}`

**Error recovery:** IF `memory` MCP server is unavailable, THEN write findings to `{{REQUIREMENTS_DIR}}/.pipeline-state.md` as a fallback and continue.

**Step 25: Confirm & Suggest Next Steps**

Present completion summary:
```
## Requirements Structuring Complete — Pipeline Handoff
- **Status:** Complete | Blocked | Partial
- **Project:** VaahApp
- **Root:** /Users/pk/Projects/vaah
- **Requirements Dir:** .vaahagents/requirements/
- **Corpus:** overview.md + open-gaps.md — [synced / partially synced]
- **Methodology Used:** <technique selected in Phase 1 Step 4>
- **Tech Stack:** [Summary — no version numbers]
- **Features:** [Count] total ([MVP count] MVP — dependency-ordered)
- **Quality Attributes:** [Coverage per ISO 25010 category]
- **Files Generated:** [List with paths]
- **Traceability:** [corpus → features → acceptance criteria — confirmed]
- **Manifest:** .requirements-manifest.json — [Confirmed / Regenerated / Failed]
- **Verification:** [Ladder steps 1-7 — Pass / Partial / Fail per check]
- **Memory Updated:** Yes / No / Deferred (fallback at .vaahagents/requirements/.pipeline-state.md)
- **Pipeline Handoff:** Ready for the milestones-and-tasks pipeline
  → Run: Review manifest at .vaahagents/requirements/.requirements-manifest.json,
    then generate milestones into .vaahagents/milestones-and-tasks/.
```

## Report

### Progress Tracking

Use the Claude Code todo list tool throughout. Create at Phase 1 covering all 7 phases. Mark exactly ONE phase `in_progress` at a time. Mark `completed` immediately after finishing — do not batch.

### Completion Report

After completing Phase 7, output the structured completion block from Step 25.
