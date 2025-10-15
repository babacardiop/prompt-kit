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

You’re set. This spec gives an agent everything needed to **refactor Spec-Kit into Prompt Kit** with the features, commands, safety guarantees, and language-specific conventions we defined.
::contentReference[oaicite:0]{index=0}
```
