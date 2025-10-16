Amazing — here’s a **complete, agent-ready specification** for converting **github/spec-kit** into **Prompt Kit** exactly as we defined. It’s written for a capable repo-aware agent (Cursor/Claude/Copilot) to follow step-by-step. You can paste this into the agent and let it run the work in your fork.

---

# Prompt Kit — Agent Implementation Spec (from GitHub/spec-kit)

## 0) Mission & Scope

**Mission:** Transform Spec-Kit (spec-driven dev) into **Prompt Kit** (prompt-driven generation).
**Core change:** Keep the original pipeline `/specify → /clarify → /plan → /tasks`, but change `/implement` to **emit prompts** (generation & verification) instead of code/tests. Add operational commands `/execute`, `/validate`, `/merge`, `/migrate`. Enforce four non-negotiable pillars (Logging, Traceability, Versioning, Compilation). Ensure **human-safe code** (regeneration never overwrites manual code) for **C#** and **React + TypeScript**.

**Out of scope (for now):** Cloud integrations, GUI dashboards, non-C#/TS languages (leave extensible hooks).

---

## 1) Naming & Top-level Identity

* Product name: **Prompt Kit**
* Preserve attribution to Spec-Kit in README (“Fork of GitHub Spec-Kit; prompt-driven evolution”).
* CLI executable name: `prompt-kit` (keep backward compatible alias if needed: `spec-kit` → prints deprecation note).

---

## 2) Repository Restructure (non-breaking)

Add a new `src/promptkit` namespace alongside existing code. Keep legacy code for reference but route the new behavior.

```
/ (repo root)
  README.md
  promptkit.constitution.yaml              # minimal constitution (targets/agents/style/principles)
  /src
    /promptkit
      constitution.ts
      /templates
        prompt-generation.hbs
        prompt-verification.hbs
        prompt-manifest.hbs
      /emit
        implementAsPrompts.ts              # replaces old implement behavior
      /run
        executeSeries.ts                   # /execute
        validateSeries.ts                  # /validate
        inputResolver.ts
        phaseSelector.ts
        /agents
          claude.ts                        # stub adapter
          cursor.ts                        # stub adapter
          copilot.ts                       # stub adapter
      /housekeeping
        logging.ts
        trace.ts
        versioning.ts
        compile.ts
      /fs
        fileIO.ts                          # atomic writes + archive + header inject
      /merge
        mergeSeries.ts                     # /merge (update-mode pipeline)
        promptDiff.ts
        promptUpdater.ts
        manifestUpdater.ts
        mergeLogger.ts
      /migrate
        migrateSeries.ts                   # /migrate (code regeneration)
        traceScanner.ts
        manifestDiff.ts
        promptComparer.ts
  /prompts                                 # generated at runtime by /implement
  /logs                                    # runtime logs
  /archive                                 # backups for prompts & code
```

---

## 3) Constitution (minimal)

Create `promptkit.constitution.yaml` in repo root:

```yaml
# Prompt Kit Constitution (minimal)
targets:
  - csharp
  - react-typescript
defaultAgent: claude
backupAgents:
  - cursor
  - copilot
promptStyle:
  tone: "precise and technical"
  format: "markdown-with-yaml-frontmatter"
  verificationChecklist: "checkboxes with ✅/❌ and brief rationale"
principles:
  - Logging
  - Traceability
  - Versioning
  - CompilationValidation
```

* Load this once and cache. Hard-fail `/implement`, `/execute`, `/merge`, `/migrate` if `targets` missing.

---

## 4) Data Models & Schemas

### 4.1 Prompt Definition File (`*.prompt.md`)

Frontmatter + Markdown body:

```yaml
---
id: generate-dto                  # unique within series+version
series: DTO-Synchronizer
version: 1.4.0                    # semver
type: generation | verification
executeWith: default | claude | cursor | copilot
inputs:
  - name: entityName
    type: string
  - name: sourcePath
    type: string
  - name: targetPath
    type: string
dependsOn: [generate-entity]      # optional
produces:
  - SuperNova.Server/.../BuildingDto.cs   # optional hints
manualExtension:
  pattern:
    csharp: "*.Custom.cs"
    react: "*.custom.tsx"
  strategy: preserve
---
# Prompt
<deterministic, stepwise instructions with {{entityName}} etc.>
```

### 4.2 Manifest (`manifest.yaml`)

```yaml
series: DTO-Synchronizer
version: 1.4.0
phases:
  - id: generate-entity
    type: generation
    inputs: [entityName, targetPath]
  - id: verify-entity
    type: verification
    dependsOn: [generate-entity]
  - id: generate-dto
    type: generation
    inputs: [entityName, sourcePath, targetPath]
    dependsOn: [generate-entity]
  - id: verify-dto
    type: verification
    dependsOn: [generate-dto]
```

### 4.3 Execution Log (JSON)

`/logs/YYYY-MM-DD/HH-mm-ss_<series>_<command>.json`

```json
{
  "timestamp": "2025-10-15T22:35:11Z",
  "series": "DTO-Synchronizer",
  "version": "1.4.0",
  "command": "/execute",
  "agent": "claude-3.5-sonnet",
  "phasesRun": ["generate-dto", "verify-dto"],
  "results": {
    "generate-dto": "Success",
    "verify-dto": "Success"
  },
  "inputs": {
    "generate-dto": {"entityName": "Building", "sourcePath": "...", "targetPath": "..."}
  },
  "filesCreated": [".../BuildingDto.cs"],
  "filesModified": [],
  "compileResult": "Success",
  "notes": ""
}
```

---

## 5) Pipeline & Commands

### 5.1 Keep unchanged

* `/specify` → parse user `@spec.md` (developer spec is the entry point)
* `/clarify`
* `/plan`
* `/tasks`

### 5.2 Replace `/implement` (emit prompts instead of code/tests)

**Implementer:** `src/promptkit/emit/implementAsPrompts.ts`

**Behavior:**

1. Read plan/tasks (from preceding stages).
2. For each task, decide **generation** vs **verification** prompt(s).
3. Create `/prompts/<series>/<version>/`:

   * render prompt files via Handlebars (`prompt-generation.hbs`, `prompt-verification.hbs`)
   * render `manifest.yaml` via `prompt-manifest.hbs`
4. Do **not** generate code or tests here.

**Prompt body requirements:**

* Deterministic lists/steps.
* Explicit file paths.
* Tie to `targets` (C# or React+TS) for style/syntax.
* Reference Human-Safe rules (see §8) in instructions.
* Verification prompts must output ✅/❌ checklist.

**Template fragments (examples):**

`prompt-generation.hbs` (C# DTO)

```hbs
---
id: {{id}}
series: {{series}}
version: {{version}}
type: generation
executeWith: default
inputs:
  - name: entityName
    type: string
  - name: sourcePath
    type: string
  - name: targetPath
    type: string
manualExtension:
  pattern:
    csharp: "*.Custom.cs"
    react: "*.custom.tsx"
  strategy: preserve
---
# Prompt: Generate DTO (C#)

1) Create/overwrite file at {{targetPath}} with a partial class `{{entityName}}Dto`.
2) All generated classes are `partial`; generated methods `virtual`.
3) Also create a companion file `{{entityName}}Dto.Custom.cs` (if not exists) for developer overrides.
4) Map properties from {{sourcePath}} with correct nullability.
5) Insert the trace header exactly (Prompt Kit will verify):
   - Series: {{series}}
   - Version: {{version}}
   - Prompt ID: {{id}}
   - Agent: {{agent}}
6) Compile after write; report any errors.

Return only the final code file(s).
```

`prompt-verification.hbs` (C# DTO)

```hbs
---
id: {{id}}
series: {{series}}
version: {{version}}
type: verification
executeWith: default
inputs:
  - name: targetPath
    type: string
  - name: sourcePath
    type: string
---
# Prompt: Verify DTO (C#)

Review {{targetPath}} versus {{sourcePath}} and confirm:

- [ ] ✅/❌ All properties exist with correct types.
- [ ] ✅/❌ Nullability matches conventions.
- [ ] ✅/❌ File contains Prompt Kit trace header with Series/Version/Prompt ID.
- [ ] ✅/❌ Generated class is partial and generated methods are virtual.
- [ ] ✅/❌ Companion `*.Custom.cs` exists (or explicitly not needed).

Provide a markdown checklist with ✅/❌ and brief rationale for each line.
```

### 5.3 `/execute` — run generation + inline verification

**Implementer:** `src/promptkit/run/executeSeries.ts`

**CLI:**

```
prompt-kit /execute <series> [version]
  [--phase <id>] [--from <id>] [--to <id>] [--only a,b] [--skip x,y]
  [--interactive|--non-interactive]
  [--agent <name>] [--summary|--details]
```

**Flow:**

1. Load constitution; load `<series>/<version>/manifest.yaml`.
2. Select phases via options (default: all in order).
3. For each selected phase:

   * resolve inputs (`inputResolver`, prompt user if `--interactive`)
   * open `.prompt.md`, merge params to body
   * **Housekeeping wrapper** (`ExecuteWithGuards`):

     * Versioning: if write will overwrite, archive old file(s) into `/archive/YYYY-MM-DD/...`
     * Call agent adapter (`agents/claude|cursor|copilot.ts`), receive files
     * Trace Header injection (see §7)
     * Atomic write via `fileIO`
     * Compilation step (see §7.4)
     * Logging: write execution JSON
4. For each **generation** prompt run its linked **verification** prompt automatically (if exists).
5. Print concise summary (✅/❌ per phase).

### 5.4 `/validate` — re-run only verification prompts

**Implementer:** `src/promptkit/run/validateSeries.ts`

**CLI:**

```
prompt-kit /validate <series> [version]
  [--phase <id>] [--only a,b] [--skip x,y] [--summary|--details]
```

**Flow:** load manifest, filter `type: verification`, run with current files, log and summarize.

### 5.5 `/merge` — evolution mode of `/specify`

**Implementers:**

* Orchestrator: `src/promptkit/merge/mergeSeries.ts`
* Diff & updates: `promptDiff.ts`, `promptUpdater.ts`, `manifestUpdater.ts`
* Log: `mergeLogger.ts`

**CLI:**

```
prompt-kit /merge <series> [version] @newSpecs.md
```

**Behavior:**

1. Load existing `manifest.yaml` & prompts (current version).
2. Run the *full pipeline* on `@newSpecs.md`: `/clarify → /plan → /tasks`.
3. Enter `/implement` **update mode**:

   * **Version bump** (semantic): 1.3.7 → 1.4.0 (configurable)
   * **Backup** current version to `/archive/<series>/<oldVersion>/`
   * Reuse unchanged prompts; modify/add only impacted prompts (use IDs)
   * Emit **new** `manifest.yaml` into `<newVersion>/`
   * Inject “Merged From vX + @spec.md” header into updated prompts
4. Optional: dry `/execute` subset (syntax correctness), log result
5. Write merge summary log.

### 5.6 `/migrate` — upgrade previously generated code to new prompt version

**Implementers:**

* Orchestrator: `src/promptkit/migrate/migrateSeries.ts`
* Trace scanning: `traceScanner.ts`
* Diff helpers: `manifestDiff.ts`, `promptComparer.ts`

**CLI:**

```
prompt-kit /migrate <series> --from <oldVersion> --to <newVersion>
  [--scope all|path:<glob>|phase:<id>] [--interactive]
```

**Behavior:**

1. Scan repo for files with Prompt Kit trace headers of `<series>` & `<oldVersion>`.
2. Map files back to prompt IDs.
3. For each, load new version prompt ID & inputs (reusing cached values; prompt for changes).
4. Regenerate files (housekeeping applies), compile, log.
5. Only touch **generated files** (never manual files; see §8).

---

## 6) Phase Selection Utilities

`phaseSelector.ts`:

* Compute ordered list of phases from `manifest.yaml`.
* Apply `--phase`, `--from/--to`, `--only`, `--skip`.
* Maintain dependency sanity (warn if selection violates dependencies).

`inputResolver.ts`:

* Gather required inputs from frontmatter.
* Interactive prompts (default) or load from `.promptkit/history.json`.
* Cache last used per `<series>/<version>/<promptId>`.

---

## 7) Housekeeping (Non-Negotiable Pillars)

### 7.1 Logging (`logging.ts`)

* Write one log per command run + optional per-phase entry.
* Structure as in §4.3.
* Location: `/logs/YYYY-MM-DD/HH-mm-ss_<series>_<command>.json`
* Include compile result summary & file lists.

### 7.2 Traceability (`trace.ts`)

* Inject file header **on every generated/overwritten file**.
* C# header example:

```csharp
// <auto-generated>
// This file was generated by Prompt Kit (Prompt-Driven Generation).
// </auto-generated>
// Created on: 2025-10-15 23:04:00
// Series: {{series}}
// Version: {{version}}
// Prompt ID: {{promptId}}
// Agent: {{agent}}
// Source Spec: {{sourceSpec}}   // optional if known
```

* TS/TSX header example:

```ts
/** 
 * <auto-generated> Prompt Kit (Prompt-Driven Generation) </auto-generated>
 * Created on: 2025-10-15 23:04:00
 * Series: {{series}}
 * Version: {{version}}
 * Prompt ID: {{promptId}}
 * Agent: {{agent}}
 * Source Spec: {{sourceSpec}}
 */
```

### 7.3 Versioning (`versioning.ts`)

* Before overwriting any file, save previous to:
  `/archive/YYYY-MM-DD/<relativePath>.<yyyyMMddHHmmss>.bak`
* Prompts versions: on `/merge`, copy entire `<series>/<version>` to archive then emit new `<newVersion>/`

### 7.4 Compilation (`compile.ts`)

* For C#: run `dotnet build --no-restore` on detected solution/project (configurable).
* For React/TS: run `tsc --noEmit` when applicable (configurable).
* On failure:

  * mark log `compileResult: "Failed"`
  * leave files in place but write a **Failure Report** with errors
  * do **not** overwrite manual files; never auto-delete outputs

---

## 8) Human-Safe Code Architecture

**Goal:** Developers can extend AI scaffolds without fear of losing changes during `/merge` or `/migrate`.

### 8.1 C# Rules

* All generated classes: `partial`
* All generated methods: `virtual`
* Create or respect companion partial files: `*.Custom.cs`
* `/migrate` ONLY touches generated base file (`*.cs`); **never** touch `*.Custom.cs`

**Example generation policy in prompts:**

* Generate `BuildingDto.cs` (base; overwrite allowed)
* Ensure `BuildingDto.Custom.cs` exists (create once; never overwrite)

### 8.2 React + TypeScript Rules

* Generated files end with `.generated.ts` / `.generated.tsx`
* Developer extension files end with `.custom.ts` / `.custom.tsx`
* Index or wrapping exports can compose both
* `/migrate` **only** overwrites `.generated.*` files

### 8.3 Prompt metadata

Include in all generation prompts:

```yaml
manualExtension:
  pattern:
    csharp: "*.Custom.cs"
    react: "*.custom.tsx"
  strategy: preserve
```

### 8.4 Conflict reporting

If manual file appears to duplicate generated symbol names, log a warning in `/logs`.

---

## 9) Agent Adapters

Create thin adapters in `src/promptkit/run/agents/`:

* `claude.ts` (primary, can be stubbed with local mock if no key)
* `cursor.ts` (shell out or file-based instruction if Cursor isn’t scriptable)
* `copilot.ts` (placeholder; document manual run if API not available)

Adapters implement:

```ts
export interface AgentAdapter {
  name: 'claude'|'cursor'|'copilot';
  runPrompt(promptText: string, context: AgentContext): Promise<AgentResult>;
}

export type AgentResult = {
  files: Array<{ path: string; content: string }>;
  notes?: string;
};
```

Adapter choice: `executeWith` in prompt or `defaultAgent` from constitution; CLI `--agent` overrides.

---

## 10) Environment & Configuration

* Use ENV for agent keys:

  * `PROMPTKIT_AGENT` (default override)
  * `CLAUDE_API_KEY`, etc.
* Local `.env` supported (gitignored).
* Constitution file required; provide defaults for build commands (paths can be configured).

---

## 11) Acceptance Criteria (must pass)

1. **/implement emits prompts only**

   * No code/tests created by `/implement`
   * Prompts contain valid frontmatter & deterministic bodies
   * `manifest.yaml` coherent (phases, dependencies)

2. **/execute works**

   * Runs generation phases; asks for inputs in interactive mode
   * Creates/overwrites **generated files only**
   * Injects trace headers
   * Archives replaced files
   * Runs compilation and logs results
   * Immediately runs verification prompts

3. **/validate works**

   * Re-runs verification prompts without generation
   * Logs results; can run subsets

4. **/merge works**

   * Given new specs, produces a **new version** of prompt series
   * Backs up old version
   * Only modifies relevant prompt files
   * Emits new manifest
   * Optional compile validation (dry run)
   * Logs changes (added/modified/removed)

5. **/migrate works**

   * Detects old-version generated files via headers
   * Re-generates with new version prompts
   * Preserves manual files (`*.Custom.cs`, `.custom.tsx`)
   * Archives old files; logs compile results

6. **Human-Safe rules enforced**

   * C#: partial classes, virtual methods, `.Custom.cs` respected
   * TS/TSX: `.generated.*` vs `.custom.*` split enforced

7. **CLI options honored**

   * `--phase`, `--from/--to`, `--only`, `--skip`, `--interactive`, `--agent`

---

## 12) Test Plan (suggested automation)

* **Unit tests** for:

  * `phaseSelector` (all combinations)
  * `inputResolver` (interactive & history cache)
  * `trace` header injection
  * `versioning` archive path creation
  * `compile` adapters (mocked)
  * `manifest` validation

* **Integration tests** (local, no API calls):

  * Fake agent adapter returning canned files (C# DTO & TS component)
  * `/implement` generates prompts
  * `/execute` with mock agent produces files → assert headers, archives, compile adapter called
  * `/validate` runs verification prompts (mock results)
  * `/merge` with new specs updates prompt version & files
  * `/migrate` regenerates code from mock v1→v2 and preserves manual files

---

## 13) Developer Experience (DX)

* `README.md` quickstart:

  ```
  prompt-kit /specify @MySpecs.md
  prompt-kit /clarify
  prompt-kit /plan
  prompt-kit /tasks
  prompt-kit /implement
  prompt-kit /execute DTO-Synchronizer 1.0.0
  prompt-kit /validate DTO-Synchronizer 1.0.0
  prompt-kit /merge DTO-Synchronizer 1.0.0 @add-edit-dto.md
  prompt-kit /migrate DTO-Synchronizer --from 1.0.0 --to 1.1.0
  ```
* `.env.sample` with agent keys.
* Example `/prompts/ExampleSeries/1.0.0/*` checked in as reference.

---

## 14) Error Handling & Edge Cases

* Missing constitution → fail with actionable message.
* Unknown series/version → list available ones.
* Invalid manifest (cycles, missing phase) → fail with diagnostic.
* Agent failure/timeout → log failure; stop or continue based on `--continue-on-error` (future flag).
* Compile failure → mark failure, do not roll back automatically; leave backups to compare.
* Manual files missing (e.g., `.Custom.cs` expected) → warn, create stub once, never overwrite.

---

## 15) Security & Compliance

* Never commit secrets.
* Logs may contain file paths & prompt inputs; do not include credential values.
* Provide `--redact-input-keys key1,key2` flag (optional backlog) for sensitive inputs.

---

## 16) Rollout Steps (for the agent)

1. Create `promptkit.constitution.yaml`.
2. Add `src/promptkit/*` per structure above.
3. Wire new CLI commands:

   * Point `/implement` to `implementAsPrompts.ts`.
   * Add `/execute`, `/validate`, `/merge`, `/migrate`.
4. Add Handlebars templates (`/templates`).
5. Implement housekeeping services.
6. Implement mock agent adapter (no real API).
7. Update README + examples.
8. Add basic tests (unit + integration with mock adapter).
9. Run dry demo: sample specs → prompts → execute (mock) → validate → merge → migrate.
10. Push branch & open PR notes with “Prompt Kit mode” changes.

---

## 17) Sample Prompts (ready-to-render)

**C# DTO — generation**

````md
---
id: generate-dto
series: DTO-Synchronizer
version: 1.4.0
type: generation
executeWith: default
inputs:
  - name: entityName
    type: string
  - name: sourcePath
    type: string
  - name: targetPath
    type: string
manualExtension:
  pattern:
    csharp: "*.Custom.cs"
    react: "*.custom.tsx"
  strategy: preserve
---
# Prompt: Generate C# DTO

Create/overwrite `{{targetPath}}` with a **partial** class `{{entityName}}Dto`.
- All generated methods must be **virtual**.
- If `{{entityName}}Dto.Custom.cs` does not exist, create an empty partial class file next to it; never overwrite a `.Custom.cs` file.

Map properties from `{{sourcePath}}` with correct nullability.
Insert the following header at the top (verbatim):

```csharp
// <auto-generated>
// This file was generated by Prompt Kit (Prompt-Driven Generation).
// </auto-generated>
// Created on: {{timestamp}}
// Series: {{series}}
// Version: {{version}}
// Prompt ID: {{id}}
// Agent: {{agent}}
````

Return the final file content(s).

````

**C# DTO — verification**

```md
---
id: verify-dto
series: DTO-Synchronizer
version: 1.4.0
type: verification
executeWith: default
inputs:
  - name: targetPath
    type: string
  - name: sourcePath
    type: string
---
# Prompt: Verify C# DTO

Review `{{targetPath}}` vs `{{sourcePath}}`:

- [ ] ✅/❌ Properties exist with correct types & nullability.
- [ ] ✅/❌ File contains the exact Prompt Kit header with correct Series/Version/Prompt ID.
- [ ] ✅/❌ Class is `partial`; generated methods are `virtual`.
- [ ] ✅/❌ Companion `*.Custom.cs` exists (or not required).

Return a markdown checklist with ✅/❌ and one-line rationale per item.
````

**React TS Component — generation**

````md
---
id: generate-component
series: UI-Layer
version: 1.0.0
type: generation
executeWith: default
inputs:
  - name: componentName
    type: string
  - name: targetDir
    type: string
---
# Prompt: Generate React Component (TypeScript)

Create/overwrite `{{targetDir}}/{{componentName}}.generated.tsx` with a functional component.
If `{{componentName}}.custom.tsx` does not exist, create an empty file that imports and composes the generated one; never overwrite `.custom.tsx`.

Add TS header:

```ts
/** 
 * <auto-generated> Prompt Kit </auto-generated>
 * Created on: {{timestamp}}
 * Series: {{series}}
 * Version: {{version}}
 * Prompt ID: {{id}}
 * Agent: {{agent}}
 */
````

Return both file contents if created/updated.

```

---

## 18) Architectural Decisions & Clarifications

### 18.1 Agent Execution Model (Critical)

**How `/execute` works:**

`/execute` is a **slash command** (like `/specify`, `/plan`) that acts as a smart prompt invoker.

**Flow:**
1. User types in agent chat: `/execute DTO-Synchronizer --phase generate-dto`
2. The `/execute` command (defined in `.claude/commands/execute.md`, etc.):
   - Reads `/prompts/DTO-Synchronizer/1.0.0/manifest.yaml`
   - Resolves inputs interactively (with history caching)
   - References the prompt file: `@prompts/DTO-Synchronizer/1.0.0/generate-dto.prompt.md`
   - Agent executes the prompt with parameters
3. Agent generates code files
4. `/execute` command automatically applies **all housekeeping**:
   - Injects trace headers
   - Archives overwritten files
   - Writes execution log
   - Runs compilation validation

**Key principle:** `/execute` is end-to-end. Prompt invocation + file generation + housekeeping happen in one command.

**Agent integration:** Uses agent's file reference capability (`@filename.prompt.md` syntax) to load and execute prompts.

### 18.2 Git & Directory Structure

**Prompts are committed to git in feature branches:**

```
Branch: 001-dto-synchronizer

/specs/001-dto-synchronizer/
  spec.md
  plan.md
  tasks.md

/prompts/DTO-Synchronizer/
  /1.0.0/
    manifest.yaml
    generate-dto.prompt.md
    verify-dto.prompt.md
  /1.1.0/
    manifest.yaml
    ...

/.claude/commands/
  specify.md
  execute.md
  merge.md
  ...

/logs/           ← gitignored
/archive/        ← gitignored
```

**Versioning strategy:**
- Prompt series versions (1.0.0, 1.1.0) committed to git
- Execution logs and archives gitignored (local only)
- Each version folder is immutable once executed

### 18.3 Multi-Language Support

**Auto-detection from file extension:**

```yaml
# No explicit language declaration needed
# Inferred from targetPath:
*.cs       → C# rules (partial classes, virtual methods, *.Custom.cs)
*.tsx      → React rules (*.generated.tsx, *.custom.tsx)
*.ts       → TypeScript rules (*.generated.ts, *.custom.ts)
*.sql      → SQL (no special split, manual management)
```

**SQL handling:**
- Generate SQL files as-is
- No `.generated.sql` / `.custom.sql` split
- Too complex to preserve manual SQL changes safely
- Users manage SQL migrations manually

**Supported languages (initial):**
- C# (partial classes, virtual methods)
- React + TypeScript (generated/custom split)
- SQL (basic generation, no preservation guarantees)

### 18.4 Input Resolution

**Interactive prompts with history caching:**

```bash
/execute DTO-Synchronizer --phase generate-dto

# Agent prompts for inputs:
> Enter entityName (last: Building): _
> Enter sourcePath (last: Server/Models/Building.cs): _
> Enter targetPath (last: Server/DTOs/BuildingDto.cs): _

# History cached in:
.promptkit/history.json
{
  "DTO-Synchronizer": {
    "1.0.0": {
      "generate-dto": {
        "entityName": "Building",
        "sourcePath": "Server/Models/Building.cs",
        "targetPath": "Server/DTOs/BuildingDto.cs",
        "timestamp": "2025-10-15T22:00:00Z"
      }
    }
  }
}
```

**Optional override:**
```bash
/execute DTO-Synchronizer --phase generate-dto --input entityName=Room --input targetPath=Server/DTOs/RoomDto.cs
```

### 18.5 Compilation Validation

**Auto-detect project from generated file:**

1. Generate file: `Server/DTOs/BuildingDto.cs`
2. Scan upward for `.csproj` or `.sln`
3. Run: `dotnet build <detected-project> --no-restore`
4. Log result to execution log
5. On failure: stop, leave files in place, show errors

**For TypeScript:**
1. Generate file: `client/src/components/BuildingCard.generated.tsx`
2. Scan upward for `package.json` with `typescript` dependency
3. Run: `npx tsc --noEmit` or `npm run type-check` (if script exists)
4. Log result

**Note:** Might compile unrelated files. Future enhancement: scope compilation to specific files.

### 18.6 Version Bumping Strategy (/merge)

**Automatic semantic versioning based on change analysis:**

```bash
/merge DTO-Synchronizer 1.0.0 @add-edit-dto.md

# Analysis:
- 3 phases added (generate-edit-dto, verify-edit-dto, generate-repo)
- 1 phase modified (generate-dto - added property)
- 0 phases removed

# Decision:
Added phases → MINOR bump: 1.0.0 → 1.1.0

# Optional override:
/merge DTO-Synchronizer 1.0.0 @add-edit-dto.md --version 2.0.0
```

**Rules:**
- Added phases → minor bump (1.0.0 → 1.1.0)
- Modified phases only → patch bump (1.0.0 → 1.0.1)
- Removed phases → major bump (1.0.0 → 2.0.0)
- `--version X.Y.Z` overrides automatic decision

### 18.7 Migration Strategy (/migrate)

**Smart diff by default, interactive review optional:**

```bash
# Default: Smart diff
/migrate DTO-Synchronizer --from 1.0.0 --to 1.1.0

# Compare prompts:
- generate-dto: prompt changed → regenerate BuildingDto.cs
- verify-dto: unchanged → skip
- generate-edit-dto: new phase → generate new files

# Preserve manual files:
- BuildingDto.Custom.cs → never touched
- BuildingCard.custom.tsx → never touched

# Interactive mode:
/migrate DTO-Synchronizer --from 1.0.0 --to 1.1.0 --interactive

Phases changed:
  ✓ generate-dto (modified) - regenerate? [y/N]: y
  ✓ verify-dto (unchanged) - skip
  + generate-edit-dto (new) - generate? [Y/n]: Y
```

**Safety guarantees:**
- Only regenerate files where prompt actually changed
- Never touch manual extension files (*.Custom.cs, *.custom.tsx)
- Archive old generated files before overwriting

### 18.8 Error Recovery

**Stop on error by default, with optional continue mode:**

```bash
/execute DTO-Synchronizer

Phase 1: generate-entity ✅
Phase 2: verify-entity ✅
Phase 3: generate-dto ❌ Compilation failed
  Error: CS0246: The type 'Building' could not be found

Execution stopped. Fix errors and re-run:
  /execute DTO-Synchronizer

# Optional: Continue on errors
/execute DTO-Synchronizer --continue-on-error

Phase 3: generate-dto ❌ Compilation failed
Phase 4: verify-dto ⚠️ Skipped (depends on phase 3)
Phase 5: generate-controller ✅

Summary: 2 succeeded, 1 failed, 1 skipped
```

**Flags:**
- Default: `--stop-on-error` (halt on first failure)
- Optional: `--continue-on-error` (mark failed, continue)
- Optional: `--skip-compilation` (skip build validation)

**No explicit resume:** Re-running `/execute` is idempotent (checks trace headers, skips completed phases).

### 18.9 Constitution Hierarchy

**Hierarchical with merge/override:**

```yaml
# Root: /promptkit.constitution.yaml
targets: [csharp, react-typescript, sql]
defaultAgent: claude
compilation:
  csharp:
    command: "dotnet build"
    
# Server override: /server/promptkit.constitution.yaml
extends: ../promptkit.constitution.yaml
targets: [csharp, sql]  # override
compilation:
  csharp:
    command: "dotnet build Server.sln --no-restore"  # override
    
# Client override: /client/promptkit.constitution.yaml
extends: ../promptkit.constitution.yaml
targets: [react-typescript]  # override
```

**Resolution:**
1. Search upward from current directory
2. Load all constitutions in path
3. Merge with child overriding parent

### 18.10 Prompt Template Distribution

**Downloaded from GitHub releases (like current Spec Kit):**

```bash
# During project init:
prompt-kit init my-project --ai claude

# Downloads from GitHub releases:
- prompt-kit-templates-claude-v1.0.0.zip
  └── .claude/commands/
      specify.md
      execute.md
      merge.md
      ...

# Templates bundled in release:
- templates/
    csharp/
      prompt-generation.hbs
      prompt-verification.hbs
    react/
      prompt-generation.hbs
      ...
```

**Customization:**
- User can modify templates locally
- `/implement` reads from project templates first, falls back to bundled

### 18.11 Agent Command Files

**Yes, continue generating agent-specific command files:**

```
/.claude/commands/
  specify.md          ← keep
  clarify.md          ← keep
  plan.md             ← keep
  tasks.md            ← keep
  implement.md        ← MODIFIED (generates prompts)
  execute.md          ← NEW
  validate.md         ← NEW
  merge.md            ← NEW (experimental flag)
  migrate.md          ← NEW (experimental flag)

/.cursor/commands/
  [same structure]

/.gemini/commands/
  [same in TOML format]
```

**Backward compatible:** Existing commands work as before.

### 18.12 Discovery Commands

**Both CLI and agent slash commands:**

**CLI (terminal):**
```bash
prompt-kit list                        # all series in project
prompt-kit show DTO-Synchronizer       # versions & phases
prompt-kit history DTO-Synchronizer    # execution logs
prompt-kit diff DTO-Synchronizer 1.0.0 1.1.0  # compare versions
```

**Agent slash commands (in chat):**
```
/promptkit.list
/promptkit.show DTO-Synchronizer
/promptkit.history DTO-Synchronizer 1.0.0
/promptkit.diff DTO-Synchronizer 1.0.0 1.1.0
```

### 18.13 Log Management

**Manual cleanup (simple approach):**

```
/logs/              ← gitignored
  2025-10-15/
    09-30-45_DTO-Synchronizer_execute.json
    10-15-22_DTO-Synchronizer_validate.json
  2025-10-16/
    ...
```

**No automatic deletion.** User manages logs manually:
```bash
# User can delete old logs:
rm -rf logs/2025-09-*

# Or keep for audit trail
```

**Future enhancement:** Add retention policy in constitution.

### 18.14 Backward Compatibility

**Clean break - Prompt Kit is a new tool:**

```bash
# Old tool (maintained separately):
specify init my-project
specify check

# New tool:
prompt-kit init my-project
prompt-kit check
```

**No automatic migration path.** Users choose when to adopt Prompt Kit.

**Rationale:** Architecture is fundamentally different. Clean separation avoids confusion.

### 18.15 Critical Test Scenarios (Must Pass for v1.0)

**All 4 scenarios must work:**

**Scenario A: Basic Round-Trip**
```bash
/specify @simple-dto.md
/plan
/tasks
/implement                # creates prompts
/execute DTO-Sync 1.0.0   # generates files
/validate DTO-Sync 1.0.0  # verifies

Checks:
✓ Prompts created with valid frontmatter
✓ Files generated with trace headers
✓ Archives created for overwritten files
✓ Logs written to /logs/
✓ Compilation ran and passed
✓ Verification checklist complete
```

**Scenario B: Multi-Language Full-Stack**
```bash
# Spec includes C# + React + SQL
/implement                # creates 15 phases
/execute FullStack 1.0.0  # generates all

Files created:
✓ Building.cs (entity)
✓ BuildingDto.cs (partial class)
✓ BuildingDto.Custom.cs (empty partial)
✓ BuildingController.cs (partial class)
✓ BuildingCard.generated.tsx
✓ BuildingCard.custom.tsx (empty)
✓ 001-create-building-table.sql

Manual edits:
User adds validation to BuildingDto.Custom.cs
User adds styling to BuildingCard.custom.tsx

Re-execute:
/execute FullStack 1.0.0
✓ Manual files preserved
✓ Generated files updated
```

**Scenario C: Evolution & Migration**
```bash
# Working v1.0.0
/execute DTO-Sync 1.0.0
✓ Files generated

# Add features
/merge DTO-Sync 1.0.0 @add-edit-feature.md
✓ v1.1.0 created
✓ Old prompts archived
✓ New prompts added
✓ Modified prompts updated
✓ Manifest updated

# Upgrade code
/migrate DTO-Sync --from 1.0.0 --to 1.1.0
✓ Smart diff identifies changed prompts
✓ Regenerates only affected files
✓ Preserves manual extensions
✓ Archives old files
✓ Compilation passes
```

**Scenario D: Error Recovery**
```bash
/execute DTO-Sync 1.0.0

Phase 5 generates invalid C#:
✓ Compilation fails
✓ Execution stops
✓ Error logged to /logs/
✓ Archives preserved
✓ Clear error message shown

User fixes:
Manual edit to BuildingDto.cs

Re-run:
/execute DTO-Sync 1.0.0
✓ Detects fixed file
✓ Continues execution
✓ Success
```

### 18.16 Implementation Phases (Feature Flags)

**Ship all commands, mark complex ones as experimental:**

**v1.0 Release:**
```bash
prompt-kit /specify          # ✅ stable
prompt-kit /clarify          # ✅ stable
prompt-kit /plan             # ✅ stable
prompt-kit /tasks            # ✅ stable
prompt-kit /implement        # ✅ stable
prompt-kit /execute          # ✅ stable
prompt-kit /validate         # ✅ stable
prompt-kit /merge            # ⚠️ experimental
prompt-kit /migrate          # ⚠️ experimental
```

**Experimental warnings:**
```bash
/merge DTO-Synchronizer 1.0.0 @new-specs.md

⚠️  WARNING: This command is experimental and may change in future versions.
    Backup your prompts before proceeding.
    
Continue? [y/N]:
```

**Graduation criteria:**
- All test scenarios pass
- Used in at least 3 real projects
- No critical bugs for 2 weeks
- Remove experimental flag in v1.1 or v1.2

### 18.17 Out of Scope for v1.0

**Explicitly NOT included in initial release:**

- ❌ Cloud integrations (Azure DevOps, GitHub Actions)
- ❌ GUI dashboards or web interface
- ❌ Languages beyond C#/React/SQL (Python, Go, Java)
- ❌ Automatic prompt optimization
- ❌ AI-powered prompt generation (prompts are still manually authored)
- ❌ Team collaboration features (conflicts, locks)
- ❌ Advanced analytics (usage metrics, quality scores)
- ❌ Plugin system for custom languages
- ❌ Automatic log retention/cleanup
- ❌ Distributed execution (multi-machine)

**Extensibility hooks provided:**
- Constitution can specify new targets
- Agent adapters are pluggable
- Template system is customizable
- Housekeeping modules are isolated

---

## 19) Updated Acceptance Criteria (with clarifications)

All criteria from §11 remain valid, with these additions:

**Additional criteria:**

8. **Multi-language support verified**
   - C# + React + SQL in single series
   - Correct file patterns applied per language
   - Manual extensions preserved per language rules

9. **Interactive input resolution works**
   - History cached to `.promptkit/history.json`
   - Previous values suggested
   - Override via `--input` flag supported

10. **Error modes handled**
    - Stop on error (default)
    - Continue on error (optional)
    - Clear error messages with recovery instructions

11. **Discovery commands work**
    - CLI: `prompt-kit list`, `show`, `history`
    - Agent: `/promptkit.list`, `/promptkit.show`, etc.

12. **Constitution hierarchy respected**
    - Root + override constitutions
    - Merge strategy correct
    - Compilation commands per project

13. **Agent command files generated**
    - All commands available as slash commands
    - Per-agent format correct (.md vs .toml)
    - Experimental flags shown in command descriptions

14. **All 4 critical scenarios pass**
    - Scenario A: Basic round-trip
    - Scenario B: Multi-language full-stack
    - Scenario C: Evolution & migration
    - Scenario D: Error recovery

---

You're set. This spec gives an agent everything needed to **refactor Spec-Kit into Prompt Kit** with the features, commands, safety guarantees, and language-specific conventions we defined.
