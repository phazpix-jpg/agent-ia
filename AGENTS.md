# VAPER — Execution engine (AI-agnostic)

This project uses the **V.A.P.E.R.** cycle (Validate → Analyze → Plan → Execute → Review).
The orchestration is driven by the Node CLI; **the AI is the executor of each step**. It works
the same in Claude Code, OpenCode and GitHub Copilot — the engine is the same.

> 🏛️ **Before any task (especially improvements), consult the [Golden Rules](.vaper/GOLDEN-RULES.md)**
> — the "Constitution" of VAPER (agent vs lib, Core vs Profile, create≠execute, etc.). A change that
> contradicts a golden rule must not be implemented without first confirming with the user.

> 📄 **Key conventions added recently:**
> - **Rule 16 (fs-type):** Every feature task carries an `fs-type` field (`configuration`, `development`,
>   `integration`, `report`) extracted from the FS. This adjusts the flow (e.g., `configuration` features
>   may skip ABAP code generation).
> - **Rule 17 (auto artifact chain):** Every `.md` deliverable (FS.md, TP.md) automatically triggers
>   `.docx` (via `md2docx`) → `.pdf` (via `md2pdf`) generation in the same sub-task — no separate step.
>   The old sub-task "Generate FS.pdf" is eliminated.
> - **Rule 18 (urgent task protocol):** Urgent tasks record the paused context, execute immediately,
>   and resume the original task afterward.
> - **Rule 19 (refinement adjustments history):** Every adjustment round is recorded in a
>   `## Refinement Adjustments` table in the main task.
> - **Rule 20 (task status journal):** `start`/`finish`/`advance` log to `.vaper/metrics/task-journal.jsonl`.
>   Use `npm run task journal` to check what was in progress after a crash.
> - **Rule 21 (lib → agent only):** Default is every capability starts as an agent. A lib is pure
>   (no logging/decisions). The agent orchestrates: call lib → validate → log → decide.
>   Exception: trivially self-validating operations (checksum, content-type check) can skip the agent.
>   Each lib SHOULD have a corresponding agent (e.g. `lib/fs-converter.js` → `@fs-converter`).
> - **Rule 22 (checkpoints & resume):** Every status change (`start`/`finish`/`next`) auto-writes a **rich
>   checkpoint** (status/phase, artifacts detected on disk, last/next step) next to the feature in
>   `in-development/Feature_<id>/checkpoints/` (and execution **logs** go to
>   `in-development/Feature_<id>/logs/` via `@task-logger`). Both are **gitignored** (temporary
>   debugging/recovery — dropped with the feature); only the **global** `task-journal` stays in
>   `.vaper/metrics/`. To **save** a manual checkpoint with your own note:
>   `npm run task save <id> -- --note "..."` (alias of `checkpoint`). To **pick a feature back up** in a
>   new session / after a crash: `npm run task resume <id>` — it prints progress, the last checkpoint
>   (artifacts + context note), recent journal activity and the next step. **A fresh session reconstructs
>   state from disk (tasks/journal/checkpoint), it does NOT remember the chat** — so put anything the
>   engine can't infer into the checkpoint note.
> - **Rule 24 (work item folders):** Artifacts live in **`in-development/<Type>_<id>/`** — the
>   workstream is **NOT** part of the path (it stays a task metadata field, and is what selects the
>   profile). `<Type>` is the Azure DevOps work item type (`Feature`, `Asset`, `Product Backlog Item`…),
>   recorded on the task as `WI-Type:` and set at creation with `--wi-type "Asset"`; **absent → `Feature`**,
>   which every pre-existing folder uses. **`done/<Type>_<id>/`** is the reserved destination for
>   finished items — the folder and the convention exist, but **nothing moves there automatically yet**;
>   path resolution reads `in-development/` first and falls back to `done/`. All path building goes
>   through `lib/feature-paths.js` (`featureDir`/`featureSubDir`/`findFeatureDir`) — never concatenate
>   the folder by hand.
> - **Rule 25 (no customer data in the repo):** Workstreams, countries, stream spellings and the
>   "global zone" code are the CUSTOMER's vocabulary and live in **`.vaper/organization.json`**
>   (gitignored; model: `organization.exemplo.json`), read through `lib/organization.js`. A clone with
>   no such file falls back to a neutral set (`WS1..WS3` / `GLOBAL`) and still runs. The same rule bans
>   real work item ids, real FS/TP titles, real Z object names and third-party personal names from
>   code, tests, fixtures and docs — use synthetic values (ids `10000NN`, `ZWSX_XX_*`, `WS1`/`WS2`).
>   Tests that need a vocabulary **inject** it (`resolveWorkstream(ws, ctx, { streams, alias })`).

> ⚠️ **npm flag pass-through:** When calling `npm run task` with flags like `--branded` or `--engine`, always use the `--` separator:  
> `npm run task md2docx 1000005 -- --branded`  
> Without `--`, npm 11+ consumes the flag as its own config and the script never sees it.

## How to run a feature from start to finish

1. The user places the FS in `backlog/` and creates the main task:
   ```
   npm run task create --id <num> --type feat --ws <WS>
   ```
2. Trigger the cascade and follow the loop:
   ```
   npm run task next <num>
   ```
3. **Engine loop** — repeat until the CLI says "✅ Ciclo VAPER concluído":
   - Run `npm run task next <num>`.
   - Read the printed block: it brings the **agent role**, the **context** (FS, templates,
     sub-tasks) and **what to write**.
   - Take on that role, produce the artifacts and update the task file.
   - Execute exactly the commands listed under **"AO TERMINAR, EXECUTE"** (they advance
     the state and re-call `next`).

## Language

- **The whole project is in English** (refactor-014): tasks (`TASKS/**`), agents, templates,
  conventions, CLI (messages/help/prompts), docs and generated artifacts (FS.md, TP.md, Readme, PR,
  SI, Review in `in-development/**`). The artifact templates in `.vaper/templates/` are in
  English — fill in the content in English too.
- **Exception — greeting/conversation:** the assistant **replies in the user's language** (daily
  ritual, PT/EN/ES); the greeting/language detection in `lib/daily-ritual.js` stays multilingual.
- See the Golden Rules (R13) in `.vaper/GOLDEN-RULES.md`.

## Rules

- Do not skip phases or invent commands: the source of truth is the output of `npm run task next`.
- Respect the gates: phase **V** blocks if the FS is incomplete; phase **R** only closes
  the feature when all sub-tasks are `done`.
- In phase **V**, **revalidate the task before proceeding**: if the problem was already solved by another
  task → `reject`/`defer`; if it changed/was partially solved → update the task and continue.
- **Confirm gate (P→E):** after planning, the plan must be **confirmed** (`npm run task confirm
  <id>`) before executing. The engine blocks entry into phase E without confirmation — applies to all.
- **Visual validation:** when the result is **not testable by code** (visual — `.docx`, image,
  diagram, layout, PDF), do not declare "done" just because the file was generated; **offer a confirmation
  task** of type **`confirm`** (ids `confirm-NNN`) for the user to validate — detailing the file to
  open, what to check and the OK criterion.
- **`confirm` tasks (human validation):** they have their **own cycle** (outside V→A→P→E→R):
  `proposed → awaiting → confirmed | rejected`. The user opens/validates with `npm run task dialog <id>` and
  decides with `dialog -- <id> --ok` (approves) or `dialog -- <id> --reject "motivo"` (rejects). Do not confuse it
  with the **command** `confirm` (the P→E plan gate) — they are distinct things.
- **`confirm` sub-task as a gate in phase E:** a sub-task of type `confirm` is completed only when
  `confirmed` (the others when `done`) and, while pending, **blocks the following sub-tasks**. The engine
  points to the validation action (`dialog`) instead of an agent. The SAP flow has **two** such gates after
  the code: **Functional Refinement** then the **DRB**.
- **Functional Refinement gate (SAP only, feat-034):** after the code and **before** the DRB, a `confirm`
  gate "Functional refinement with the analyst", driven by the **`@vaper-refiner`** agent as a **loop**: the
  analyst approves **as-is** (`dialog <id> --ok` → unblocks the DRB) or **with adjustments** (the agent records
  an adjustment-round sub-task, applies the changes to the TP + `src/`, then re-asks). `reject` is reserved for
  "refinement abandoned", never for adjustments. SAP cascade: `… → code → refinement → DRB → PR`.
- **DRB gate (SAP only):** the **DRB** gate "TP approved at DRB" comes **after** the Functional Refinement
  and before the following tasks (feat-015) — engineering approves the TP.
- **PR after the DRB (SAP only):** after the DRB gate, the SAP flow has a "Generate Pull Request document" step
  (sub-task `doc`, executed by the **`@pr-generator`** agent) — generates `docs/PR_<id>.md` (`.docx` optional via
  `md2docx`). Only in the `sap` profile; never before the DRB (feat-024).
- Sub-tasks in phase **E** are required only for `feat`/`refactor` (and SAP features); small types
  (`docs`/`fix`/`chore`/`perf`) run the main task itself in E.
- Agents live in `.vaper/agents/` and `.lib/agents/`; templates in `.vaper/templates/`.
- **Every lib has a corresponding agent.** The agent is the public interface — no code calls a lib
  directly (Rule 21). Example: `lib/docx2md.js` → agent `@docx2md`; `lib/fs-converter.js` → agent
  `@fs-converter`.
- **An external library always stays behind an adapter** (an interface of ours, swappable/mockable lib):
  see `.vaper/conventions/external-libs-adapter.md`.
- **Feature document names** (`docs/`): `<tipo>_<id>_<slug>_v<n>.<ext>` (FS/TP/Readme/PR) —
  see `.vaper/conventions/document-naming.md` (helpers in `lib/doc-naming.js`). An FS `.docm` is converted
  to `.docx` (`lib/docm-to-docx.js`). In the `backlog/` the name is the raw attachment name (collision → fix-008).
- **Core vs Profile architecture** (in progress, refactor-003): profiles in `.vaper/task-cli/profiles/`
  (`core` for ws VAPER, `sap` for the domain); the Core selects by workstream via `resolveProfileId`.
  The lists of workstreams/types/countries live in the profiles (Core derives them via `allWorkstreams`/`getProfile`).
  The **context shown in the step** (`next`/`emitStep`) also comes from the profile (`stepContext`): SAP tasks
  see FS/`mapping_rules`/ABAP Naming `Z`; `VAPER` tasks see a lean context (without that).
  The accept/per-phase/per-type step **defaults** also come from the profile (`profile.defaults`): `sap` brings
  the domain texts (SEGW/BAPI/Swagger…); `core`, those of tool improvement. The **Core became
  agnostic** (refactor-010): work-types (`profile.workTypes`), phase descriptions/criteria
  (`defaults.phaseDescription/entryCriteria/exitCriteria`) and the default sub-task agent
  (`phaseAgents.defaultExecutor`) also live in the profiles — there is no longer a domain `kind` in `index.js`.
  The **domain commands** (`fs`/`feature`/`reprocess`/`md2docx`+`tp2docx`) are **declared by the
  profile** (`profile.commands`, refactor-012); the Core resolves them via `findProfileCommand` and dispatches
  through the registry (the handler bodies remain in the Core). The Core's switch does not hardcode SAP commands.
- Document conversion: `npm run task fs <id>` reads the FS (**docx→md**, input); `npm run task
  md2docx <id|arquivo.md>` generates the deliverable (**md→docx**, output — e.g., `TP.md` → `TP.docx`).
  For **visual validation**, `npm run task md2pdf <id|arquivo.md> [--png]` renders **md→pdf** (pandoc +
  headless Chrome, behind the `md2pdf` adapter / `pandoc-chrome` engine) — the symmetric pair of `md2docx`,
  domain-agnostic (works on any `.md`: FS, TP, card). The Core registers `md2pdf` directly (not profile-bound).
  `--png` also emits `<name>.p1.png` (screenshot of the rendered first page, feat-037) — the assistant can
  **read that PNG to self-validate** the render before asking the user for a visual confirmation (R12).
- **FS moved+renamed to `docs/` when the feature is created (fix-009/feat-031):** `npm run task new --feature <id>`
  **moves** the FS from `backlog/` to `in-development/<Type>_<id>/docs/` (leaves no copy in the backlog) and
  **renames it to the canonical name** `FS_<id>_<slug>_v1.<ext>` (feat-028 convention; **rename-only**, original
  extension preserved — a `.docm` stays `.docm`; no overwrite if the canonical name already exists). This way
  the feature is **not reprocessed by mistake** and the source FS stays together with the artifacts. The locator
  `findBacklogFS` looks for the FS in `<Type>_<id>/docs/` **and** in `backlog/` (in that order), so `fs`,
  the step context and the metadata inference keep working after the move. Pure helpers in
  `lib/fs-locate.js` (`featureIdFromFsName`/`matchFsFile`). `reprocess` still returns the FS from
  `docs/`→`backlog/` to reprocess it. **A sibling FS PDF is also auto-generated (chore-009):** after the
  move, `newFeatureTask` calls the best-effort `generateSiblingPdf` (wraps the `docx2pdf` adapter / Word)
  to drop `FS_<id>_<slug>_v1.pdf` next to the `.docx` for previewing in VS Code. It **never breaks**
  feature creation — if Word is missing/non-Windows/locked it just warns; `--no-pdf` opts out.
- **Azure DevOps access:** credentials in the `.env` (`AZURE_DEVOPS_ORG`/`_PROJECT`/`_BASE_URL`/`_PAT`),
  template in `.env.example`; the PAT is a secret (`.env` in the `.gitignore`). Collected by a `confirm` task
  (fills in the `.env` → `dialog <id> --ok`). The lib `lib/devops-config.js` validates and exposes the config
  (`loadDevopsConfig()`), the base of the DevOps client/adapter (feat-018).
- **DevOps — list and download FS (feat-022):** `npm run task pbis` lists the PBIs/TPs pending for the
  user (WIQL); `npm run task fetch-fs <pbiId>` follows PBI→Feature→attachments and downloads the FS (`.docx`/`.docm`)
  to `backlog/`. Client in `lib/devops-client.js` (facade) over the reusable HTTP adapter
  `lib/http-client.js` (injectable/mockable) — convention `external-libs-adapter`.
- **DevOps — download FS in batch (feat-026):** `npm run task fetch-tps` searches your PBIs with **"TP" in the
  title** (WIQL assigned-to-me) and downloads **all** the FS to `backlog/`, with a summary at the end. Filters:
  `--state` (default `Created`; `""` = all), `--area`, `--title`. Orchestration in `lib/fetch-tps.js`
  (`fetchFsForPbi`/`runFetchTps`, injectable/mockable); `fetch-fs` reuses the same `fetchFsForPbi`.
- **Tolerant linked-Feature lookup (fix-013):** if a PBI's linked Feature is inaccessible (404 / no read
  permission), `fetchFsForPbi` no longer aborts — it catches the error, falls back to the PBI's own
  attachments, and surfaces `featureWarning` (logged by the command). It only fails when **no** FS is found
  on the PBI **nor** an accessible Feature.
- **TP-ingest — build the model from an existing hand-authored TP (feat-058):** when a feature's `docs/`
  already holds a **hand-authored** `TP_<id>_*_v<N>.md` (N≥1, excluding our own `*_from-model_*`),
  `model-map` **ingests** it: `lib/tp-ingest.js` (pure) detects the file and parses its sections
  (Feature Information, §1.1, §2.4 Components, §2.7.2 steps, Open Points) into a partial tp-model via
  `doc-mapping.routeSections`; then `mergeTpModels` merges it over the FS-derived model with **old-TP
  concrete values winning** and the FS filling the gaps (arrays merge by name/id; open points union).
  Per-field provenance is recorded under `raw.provenance` (`old-tp` | absent = fs). **No-op when no such
  TP exists** — identical output to before. Reconciles Golden Rule 23 (model = source of truth) with
  pre-existing hand-authored TPs. Free-prose sections without a table stay for the agent to enrich.
- **Decision premises — engineering guidelines annotated into the model (feat-055):** a consultable data
  base (`.vaper/conventions/decision-premises.json`) of engineering premises (Smartform/SAPscript → Adobe
  Form; DDIC view → CDS view entity; custom FM → Global Class; Dynpro → RAP/Fiori; Clean ABAP). `model-map`
  matches each TP component's type via `lib/decision-premises.js` (`applyPremises`, pure) and **annotates**
  the matches into `tp-model.raw.recommendations` (`{object, premise, recommend, reason, gate}`); preflight's
  **`decision-premises`** warning surfaces them (non-blocking). Data only — @solution-architect / the DRB
  decides adoption (each carries a `gate`). Grounded in `abap-standards.md`.
- **Learned gate-checks — the pre-validation feedback loop (feat-056):** official findings from the human
  gates (technical validation / functional refinement / DRB) are recorded with `npm run task gate-learn
  <technical|functional|drb> "<finding>" [--feature <id>] [--promote]` into `.vaper/mappings/gate-checks.json`
  (mirrors `model-learn`/`mapping-learn`: a finding becomes an **active pre-check** when it recurs across
  ≥3 distinct features or is `--promote`d). `preflight` surfaces the active **technical** checks as
  reminders (formalizing `@before-technical`) so a known issue is caught **before** the next feature reaches
  the official gate. Pure lib `lib/gate-checks.js` (`recordFinding`/`activeChecks`); the `@before-functional`
  / `@before-drb` agent prompts are a follow-up (the base already supports all three gates).
- **TP versioning per cycle — file-level `_vN` (feat-057):** each closed refinement cycle produces a NEW
  version file instead of overwriting in place. `npm run task tp-version <id> [--reason "..."]` computes the
  next `_vN` (`lib/tp-ingest.js` `nextTpVersion`, pure), renders it from the model (carry-forward — the
  model is the source of truth) and **keeps the prior versions** as frozen history; the reason tags the
  Version table (`model-render --reason` also accepts it). v0 = draft (tp-publish), v1 = technically
  validated, v2 = functional-refinement closure, etc. Bump is **per cycle at closure**, not per round
  (rounds stay in the Rule 19 Refinement Adjustments table). Reverses the old "single `_v1` in place".
- **Name collision in the backlog (fix-008):** when saving the FS, if a file with the same name already exists,
  it prefixes `Feature_<featureId>_` (resolved id) so it does **not overwrite**, warning in the summary. Without collision,
  it keeps the original name. Pure helper `uniqueBacklogName` in `lib/fetch-tps.js`.
- **DevOps — publish the TP v0 (feat-063):** `npm run task tp-publish <featureId> [--apply]` closes a feature in
  Azure DevOps: attaches `TP_*_v0.docx` on the Feature, moves the **TP-PBI** (the `[..._TP]` child of the Feature)
  to **In Refinement**, and creates the **`[ABAP] TP Design v0`** task (born `To Do` → PATCH `Done`,
  `Activity=Technical Documentation`, area/iteration/assignee inherited from the TP-PBI). **DRY-RUN by default**
  (prints the 7-step plan; sends nothing) — `--apply` executes. **Idempotent** (already-attached v0 / already
  In Refinement / existing design task → skipped), **verify-after-write** (re-reads each work item), and
  **backup-before-remove** (a replaced older TP is downloaded to `Feature_<id>/_removed_attachments/` first).
  Ownership is confirmed via WIQL `@Me`. The write ops live in the **`devops-client` adapter** (`patchWorkItem`,
  `uploadAttachment`, `createWorkItem` + pure json-patch builders) over `http-client` (`patchJson`/`postJsonPatch`/
  `postBinary`, injectable transport); the pure precondition→plan logic is `lib/tp-publish.js` (`planPublish`).
  The v0 file must be generated beforehand (`model-render`/`model-tp`).
- **Object naming (feat-066):** `npm run task model-name <id>` (also a `[1b/5]` step in `model-tp`)
  enriches `fs-model.json` `objects[]` with a convention-conforming `Z` `suggested_name` +
  `naming_status` (`valid|suggested|needs-review`), via `lib/object-naming.js` +
  `.vaper/conventions/naming-convention/`. **v0 posture:** suggest/validate, never auto-assign; standard
  objects (VBAK, A700, BAPI_*) stay as-is; undeterminable parts → `TBD`. The `@sap-object-mapper` agent
  owns the judgment (object type, descriptive text, resolving `needs-review`).
- **TP pipeline wired into the cascade (fix-019):** `npm run task model-tp <id>` is the **5-step**
  one-command pipeline — `extract → map → estimate → render → preflight` (the **estimate** fills GAP
  Complexity hours into `tp-model.json` before the render, so the branded TP reflects them). It is the
  **canonical TP step of the cascade**: the SAP `workTypes.fs` TP sub-task names `model-tp`, so a
  from-scratch feature (`task next`) ships with Classification, header and effort estimation without the
  executor having to remember the model-* commands. The pre-DRB **`--v0`** draft stays owned by
  `tp-publish` (the cascade renders the normal working TP, which freezes as v1 at technical validation).
- **TP quality guards (feat-064):** (a) **assumed-estimate marker** — when `model-estimate` defaults a
  complexity, the GAP Complexity hours render with a `***` marker + legend (in .md and branded .docx) so a
  guessed number is never mistaken for a confirmed one; (b) **ABAP implementation steps** — `implementation_steps`
  is a first-class fs-model field (`@fs-model-extractor` fills it for any ABAP-activity FS, not just forms);
  the map carries it to TP §2.7.2 Logic Description, and preflight's **`abap-steps`** check warns
  (non-blocking) when an ABAP TP has 0 steps. Preflight is now **severity-aware**: failed `warning` checks
  surface (⚠) but do not block; only critical failures block.
- **XLSX enrichment foundation (feat-076):** VAPER now has deterministic spreadsheet ingestion behind
  adapters: `lib/xlsx-reader.js` (dependency boundary) + `lib/xlsx-parser.js` (sheet/column validation)
  + `lib/xlsx-enrichment.js` (FS-model bridge). When `fs-model.json` carries `xlsx_enrichment`, `model-map`
  propagates it to `tp-model.raw.xlsx_enrichment` and can derive fallback implementation steps from tables
  only when no FS implementation steps/layout rules are present (non-disruptive to existing flows).
- **Daily ritual (feat-020):** upon receiving a **greeting** (bom dia / good morning / buenos días, in
  PT/EN/ES), **reply in the same language** and record the **initial cost snapshot**. Upon receiving a
  **farewell**, record the final snapshot and show the **day's delta** + the tasks roll-up. The `/cost`
  belongs to the **runtime** (Claude Code) and the assistant does **not** read it directly — the **source of the value is the user**
  (they run/share the cost), keeping the *tool-agnostic* behavior. Record with
  `npm run task snapshot <open|close> --cost <valor>` (writes to `.vaper/metrics/daily.jsonl`); see the
  summary with `npm run task daily`. Greeting/language detection and delta: `lib/daily-ritual.js`.
  **Greeting counters (feat-029):** the greeting includes the **first name** from `VAPER_ASSIGNEE` and
  the briefing — **Pending** (open: neither `done`/`confirmed`/`rejected`/`deferred`), **Deferred**
  (`deferred`) and **Ideas** (`parqueada`/`em análise` in `IDEAS.md`), **hiding a zeroed block** and
  counting **main tasks only**. Deterministic source: `npm run task dashboard -- --text` (the same
  pure dashboard aggregation, `lib/dashboard.js`). If the previous day's **closing** snapshot has already
  been recorded, **skip the opening** (unless explicitly requested). With nothing pending/deferred, offer to read
  the Azure DevOps PBIs (`pbis`).
- **Tasks dashboard (feat-029):** `npm run task dashboard` generates the "C" briefing as **static
  HTML** in `.vaper/metrics/dashboard.html` (gitignored) and opens it in the browser — **on demand**
  (it reads the `.md` files on the spot, **with no persisted index**: avoids the silent staleness of idea 001).
  Actions in the HTML are **passive** (copy command / open task); running the cycle continues with the
  AI agent (tool-agnostic). Flags: `--no-open` (headless/CI), `--text` (briefing to stdout).
  Pure aggregation/HTML in `lib/dashboard.js`; opening the browser behind the adapter `lib/open-browser.js`.
- **Dashboard rendered by the AI (feat-030):** when the user asks for the dashboard / "what do I have
  today" / pending items, **run `npm run task dashboard --json` and render via the template**
  `.vaper/conventions/dashboard-template.md` (instead of opening the HTML). The JSON brings, per task,
  `phase`, `planConfirmed` and `nextAction` (suggested deterministic command). Interactivity is
  **conversational**; the action boundary holds (deterministic ones yes; running the V→A→P→E→R cycle only by
  the user's explicit decision). HTML and `--text` remain as alternative paths.
- **FS-backlog + Kanban mode (feat-036):** the `--json` model also carries **`backlogFs`** — FS `.docx`
  still in `backlog/` (not yet imported via `task feature <id>`): the top of the funnel. The scan is
  **profile-driven** (the SAP profile declares `backlog = { dir, match, parse }`; the Core only reads the
  dir — Rule 2). The template adds a **Kanban mode** (columns Ideas → FS-backlog → VAPER → workstreams;
  cards sorted by `PHASE_ORDER` via `phaseRank`), an alternative layout of the same model. List/HTML/`--text`
  unchanged.
- **Kanban TUI command (feat-040):** `npm run task kanban` opens an **interactive terminal board** (same
  data as the dashboard, no index). This is an **authorized exception to Rule 13** (the "no TUI" convention):
  the TUI is a human-convenience view — the agent-agnostic path stays `dashboard --json` + AI render.
  It is **non-TTY safe**: piped/CI/agent contexts (or `--static`/`--no-tty`) get a one-shot static print,
  never a hang. Rendering goes through an adapter (`lib/kanban.js` → `lib/kanban-engines/{builtin,static}.js`,
  Rule 3); default engine is built-in (zero dependency, `readline` + ANSI).
- **CLI reports — data (CLI) separated from presentation (AI):** every command that displays **a lot of
  information** or **formatted information** (report/dashboard/briefing) must expose **`--json`** with the
  full model (source of truth, in a testable pure lib) and leave the **rich rendering to the AI**,
  which follows a **versioned template** in `.vaper/conventions/`. No TUI/input loop (depends on
  TTY, ties down the tool); interactivity is **conversational**. See
  `.vaper/conventions/cli-reports-data-vs-presentation.md` (instance in progress: dashboard `--json`,
  feat-030).
- **Tokens per activity (feat-019):** upon completing an **activity** (task/sub-task), record the consumption
  with `npm run task tokens <id> --in N --out N [--phase P]` (writes to `.vaper/metrics/tokens.jsonl`). Since
  the CLI **does not measure tokens**, the **source of the value is reported** (the assistant on completion, or the user) —
  *tool-agnostic*, without a hook or transcript parsing. The **roll-up** comes out in `npm run task tokens [--day]`
  and also appears in `daily`. Pure sum/parse: `lib/tokens.js`.
- **MCP servers as optional engines (feat-025):** external capabilities (DevOps, SAP, Mermaid) can be
  offered as **MCP server engines** alongside the current implementation (REST/subprocess). Each adapter
  exposes `{ engine: 'current' | 'mcp' }` with automatic fallback and **score tracking**
  (`lib/mcp-score.js`, logs to `.vaper/metrics/mcp-scores.jsonl`). The DevOps adapter (`devops-client.js`)
  now supports `engine: 'mcp'` (feat-032). See proposed task `feat-033` (SAP MCP). The adapter pattern
  (`external-libs-adapter.md`) already supports this multi-engine architecture.
- See `README.md` for the full CLI reference.
