# Prompt Kit - Project Summary

> **Transforming Spec Kit into a governed, auditable, AI-powered code generation system**

---

## Executive Summary

**Prompt Kit** is an evolution of GitHub's Spec Kit that transforms ad-hoc AI code generation into a **CI/CD-grade engineering process**. Instead of directly generating code, Prompt Kit generates **versioned, reusable prompt series** that produce code with full traceability, logging, and human-safe regeneration patterns.

---

## The Core Innovation

### Spec Kit (Current)
```
Specification → AI → Code (one-shot, no traceability)
```

### Prompt Kit (New)
```
Specification → Prompt Series → Versioned Execution → Governed Code
                 (reusable)      (auditable)          (regenerable)
```

---

## Four Non-Negotiable Pillars

| Pillar | What It Does | Benefit |
|--------|-------------|---------|
| 🧾 **Logging** | Every execution recorded (inputs, outputs, agent, compile results) | Reproducibility |
| 🔗 **Traceability** | Auto-generated headers in every file (series/version/agent/prompt ID) | Auditability |
| 🗂️ **Versioning** | Files archived before updates; prompt versions immutable | Rollback safety |
| 🧱 **Compilation** | Post-generation compile guarantees syntactic correctness | Trust |

---

## Command Structure

### Creation Pipeline (Spec Kit Compatible)
- `/specify` - Define what to build (requirements, user stories)
- `/clarify` - Resolve ambiguities through structured questions
- `/plan` - Create technical implementation plan with tech stack
- `/tasks` - Generate actionable task breakdown

### Prompt Generation (New)
- `/implement` - **Changed behavior**: Creates prompt series (not code)
  - Outputs: `/prompts/<series>/<version>/manifest.yaml` + `*.prompt.md` files
  - No code generated at this stage

### Operational Commands (New)
- `/execute` - Run prompt series, generate code with governance
  - Interactive input resolution
  - Automatic housekeeping (headers, archiving, compilation, logging)
- `/validate` - Re-run verification prompts (detect manual breakage)
- `/merge` - Evolve existing prompt series (version bump with new specs) ⚠️ *experimental*
- `/migrate` - Upgrade generated code from old to new prompt version ⚠️ *experimental*

### Discovery Commands (New)
- `/promptkit.list` - Show all series in project
- `/promptkit.show <series>` - Display versions & phases
- `/promptkit.history <series>` - View execution logs
- `/promptkit.diff <series> v1 v2` - Compare prompt versions

---

## Human-Safe Code Architecture

### The Problem
AI regeneration typically destroys manual code edits.

### The Solution
Language-specific patterns that separate generated from manual code:

#### C# Pattern
```csharp
// BuildingDto.cs (AI-generated, regenerable)
public partial class BuildingDto { 
  public int Id { get; set; }
  public virtual void Validate() { /* generated */ }
}

// BuildingDto.Custom.cs (manual, never touched)
public partial class BuildingDto {
  public override void Validate() {
    base.Validate();
    // Custom validation logic
  }
}
```

#### React/TypeScript Pattern
```tsx
// BuildingCard.generated.tsx (AI-generated, regenerable)
export const BuildingCard = ({ building }) => (
  <div>{building.name}</div>
);

// BuildingCard.custom.tsx (manual, never touched)
import { BuildingCard } from './BuildingCard.generated';
export const BuildingCardEnhanced = (props) => (
  <div className="custom-wrapper">
    <BuildingCard {...props} />
  </div>
);
```

**Guarantee:** `/execute` and `/migrate` **never overwrite** `.Custom.cs` or `.custom.tsx` files.

---

## Key Architectural Decisions

### Agent Integration
- **`/execute` is a slash command** (not an API call)
- Works in agent chat (Claude, Cursor, Copilot, etc.)
- References prompts via `@prompts/series/version/phase.prompt.md`
- Agent generates code, `/execute` applies housekeeping

### Git Integration
- Prompt series committed to feature branches
- Structure: `/prompts/<series>/<version>/`
- Logs and archives gitignored (local only)
- Immutable versions (1.0.0, 1.1.0, etc.)

### Multi-Language Support
- **Auto-detection** from file extension
  - `*.cs` → C# rules (partial classes, virtual methods)
  - `*.tsx` → React rules (generated/custom split)
  - `*.ts` → TypeScript rules
  - `*.sql` → SQL (no special handling)

### Version Bumping (Automatic)
- Added phases → Minor bump (1.0.0 → 1.1.0)
- Modified phases → Patch bump (1.0.0 → 1.0.1)
- Removed phases → Major bump (1.0.0 → 2.0.0)
- Override: `--version X.Y.Z`

### Error Handling
- **Default:** Stop on first error
- **Optional:** `--continue-on-error` (mark failed, continue)
- **Recovery:** Re-run `/execute` (idempotent, skips completed)

### Constitution Hierarchy
```yaml
/promptkit.constitution.yaml           # root defaults
/server/promptkit.constitution.yaml    # server overrides
/client/promptkit.constitution.yaml    # client overrides
```
Child overrides parent (search upward from current dir)

---

## Directory Structure

```
/my-project/
  promptkit.constitution.yaml
  
  /specs/001-dto-feature/
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
  
  /.claude/commands/        # Agent-specific slash commands
    specify.md
    execute.md
    merge.md (experimental)
    ...
  
  /logs/                    # Execution logs (gitignored)
    2025-10-15/
      09-30-45_DTO-Synchronizer_execute.json
  
  /archive/                 # Backup files (gitignored)
    2025-10-15/
      Server/DTOs/BuildingDto.cs.20251015093045.bak
```

---

## Critical Success Criteria (v1.0)

All **4 scenarios** must pass:

### ✅ Scenario A: Basic Round-Trip
```bash
/specify → /plan → /tasks → /implement → /execute → /validate
```
- Prompts created with valid structure
- Files generated with trace headers
- Archives, logs, compilation all working

### ✅ Scenario B: Multi-Language Full-Stack
```bash
C# backend + React frontend + SQL migrations (15 phases)
```
- All languages generate correctly
- Manual extensions (`.Custom.cs`, `.custom.tsx`) preserved
- Re-execution safe

### ✅ Scenario C: Evolution & Migration
```bash
v1.0.0 → /merge → v1.1.0 → /migrate
```
- Version bump automatic
- Old prompts archived
- Smart diff regenerates only changed files
- Manual code preserved

### ✅ Scenario D: Error Recovery
```bash
Phase 5 fails → user fixes → re-execute
```
- Compilation errors logged
- Execution stops cleanly
- Archives preserved
- Recovery works

---

## Implementation Strategy

### v1.0 Release Plan
- ✅ **Stable:** `/specify`, `/clarify`, `/plan`, `/tasks`, `/implement`, `/execute`, `/validate`
- ⚠️ **Experimental:** `/merge`, `/migrate`
  - Show warning on use
  - Require confirmation
  - Graduate after 3 real projects + 2 weeks stable

### Technology Stack
- **CLI:** Python 3.11+ (same as Spec Kit)
- **Templates:** Handlebars (for prompt generation)
- **Distribution:** GitHub releases (same as Spec Kit)
- **Agent Support:** Claude, Cursor, Copilot, Gemini, Qwen, etc.

### Backward Compatibility
- **Clean break:** Prompt Kit is a new tool (`prompt-kit` command)
- Spec Kit continues as separate tool (`specify` command)
- No automatic migration (architectures too different)

---

## What's NOT in v1.0

- ❌ Cloud integrations (Azure DevOps, GitHub Actions)
- ❌ GUI/dashboards
- ❌ Languages beyond C#/React/SQL (Python, Go, Java later)
- ❌ AI-powered prompt generation (prompts manually authored)
- ❌ Team collaboration (locks, conflicts)
- ❌ Advanced analytics
- ❌ Plugin system

**Extensibility hooks provided** for future expansion.

---

## Sample Workflow

```bash
# 1. Create specification
/specify @build-dto-layer.md

# 2. Clarify ambiguities
/clarify

# 3. Create technical plan
/plan

# 4. Generate task breakdown
/tasks

# 5. Generate prompt series (NEW!)
/implement
# Output: /prompts/DTO-Synchronizer/1.0.0/

# 6. Execute prompts to generate code (NEW!)
/execute DTO-Synchronizer 1.0.0
# Enter entityName: Building
# Enter sourcePath: Server/Models/Building.cs
# Enter targetPath: Server/DTOs/BuildingDto.cs
# ✅ Generated: BuildingDto.cs (with trace header)
# ✅ Generated: BuildingDto.Custom.cs (empty)
# ✅ Archived: (none, first run)
# ✅ Logged: /logs/2025-10-15/09-30-45_DTO-Synchronizer_execute.json
# ✅ Compiled: dotnet build Server.csproj --no-restore (Success)

# 7. Verify correctness (NEW!)
/validate DTO-Synchronizer 1.0.0
# ✅ All checks passed

# 8. Evolve with new requirements (NEW! - experimental)
/merge DTO-Synchronizer 1.0.0 @add-edit-operations.md
# Analysis: 2 phases added, 1 modified
# New version: 1.1.0

# 9. Upgrade existing code (NEW! - experimental)
/migrate DTO-Synchronizer --from 1.0.0 --to 1.1.0
# Smart diff: regenerate changed files only
# Preserve: BuildingDto.Custom.cs (manual code)
```

---

## Benefits Summary

| Aspect | Advantage |
|--------|-----------|
| **Governance** | Every AI action logged and traceable |
| **Evolution** | `/merge` and `/migrate` keep prompts and code in sync |
| **Safety** | Manual code never overwritten (language-specific patterns) |
| **Flexibility** | Partial/virtual (C#), generated/custom (React) patterns |
| **Scalability** | Granular phase execution, versioned series, CI/CD ready |
| **Auditability** | Trace headers link every file to series/version/prompt/agent |
| **Reproducibility** | Execution logs enable exact replay |
| **Rollback** | Archives + immutable versions enable safe rollback |

---

## Getting Started

### For Developers
1. Read full specification: `todo.md` (1,337 lines, implementation-ready)
2. Review architectural decisions: `todo.md` Section 18
3. Understand test requirements: `todo.md` Section 18.15

### For Implementers
1. Fork/clone `github/spec-kit` repository
2. Create `src/promptkit/` namespace alongside existing code
3. Follow rollout steps: `todo.md` Section 16
4. Implement with experimental flags for complex commands

### For Product Managers
- **Impact:** Transforms AI coding from "fingers crossed" to "auditable process"
- **Risk:** Clean break from Spec Kit (no migration path)
- **Timeline:** All 4 critical scenarios must pass before v1.0 ship
- **Adoption:** Experimental flags de-risk `/merge` and `/migrate`

---

## Next Steps

1. ✅ **Specification complete** (this document + `todo.md`)
2. ⏳ **Implementation** (follow `todo.md` sections 0-17)
3. ⏳ **Testing** (4 critical scenarios + acceptance criteria)
4. ⏳ **Documentation** (README, PROMPTKIT.md, examples)
5. ⏳ **Release** (v1.0 with experimental flags)
6. ⏳ **Validation** (3 real projects, 2 weeks stable)
7. ⏳ **Graduation** (remove experimental flags in v1.1)

---

## Links

- **Full Specification:** `todo.md` (implementation guide)
- **Concept Overview:** `overview.md` (strategic vision)
- **Source Project:** [github/spec-kit](https://github.com/github/spec-kit)

---

*Last Updated: October 16, 2025*
*Specification Status: Complete and implementation-ready*

