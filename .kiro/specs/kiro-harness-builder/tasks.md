# Implementation Plan: kiro-harness-builder

## Overview

`kiro-harness-builder` ships as a **Kiro Skill** (agent-driven Markdown orchestrator + per-phase playbooks + artifact templates) that installs a harness built from Kiro's steering / permissions / hooks layers. Because the wizard's own control logic is agent-driven Markdown, the plan splits into two tracks that are wired together at the end:

1. **A thin TypeScript reference implementation of the wizard's pure-logic units** (phase controller, scope/path resolution, confirm-before-write gate, merge engines, permission effect resolver, State_Manager, input validators, hook-schema mapper, and generated-artifact structural guards). This is the testable substrate that the Markdown playbooks describe in prose — it exists so the Correctness Properties can be verified with property-based tests (fast-check), schema validation (ajv), and shellcheck/BATS over generated scripts.
2. **The Kiro Skill artifacts** (`SKILL.md`, `references/*.md`, `templates/*`) plus repo packaging/docs, whose behavior is specified by — and kept consistent with — the reference implementation and templates.

The instruction driving this breakdown: *Convert the feature design into a series of prompts for a code-generation LLM that will implement each step with incremental progress. Make sure that each prompt builds on the previous prompts, and ends with wiring things together. There should be no hanging or orphaned code that isn't integrated into a previous step. Focus ONLY on tasks that involve writing, modifying, or testing code.*

> **Language selection:** The reference implementation and property tests are in **TypeScript** (fast-check for PBT, ajv for JSON-schema validation). Generated hook scripts are **Bash** verified with **shellcheck + BATS**, per the design's Testing Strategy.

## Tasks

- [ ] 1. Resolve Open Assumptions and scaffold the project
  - [ ] 1.1 Verify the Kiro-specific facts the design flags as unconfirmed and record findings
    - Confirm the versioned Kiro hook JSON schema (`version`, `hooks[]`, `name`/`trigger`/`matcher?`/`action{type,command|prompt}`/`timeout?`/`enabled?`) and the `PreToolUse` decision payload + exit-code contract (0 success/stdout, 2 block/stderr, other = non-blocking)
    - Confirm the permission-config on-disk location and serialization for workspace scope (the `settings/permissions.json` assumption vs. a hashed `workspace-roots/<hash>/` store) and whether it is file-editable on Kiro Web
    - Confirm the project-directory environment variable name (`KIRO_PROJECT_DIR` assumption) and its `$PWD` fallback
    - Write findings into a `docs/kiro-facts.md` note that downstream tasks read instead of hardcoding assumptions; where a fact cannot be confirmed, record the fallback the design mandates
    - _Requirements: 9.5, 7.1, 10.7_
    - _Design: Open Assumptions #1–#5_

  - [ ] 1.2 Scaffold the Kiro Skill directory structure and packaging manifest
    - Create `skills/wizard/` with empty `references/` and `templates/hooks/` and `templates/steering/` subdirectories (placeholder `.gitkeep`-style markers only where needed)
    - Author the Kiro Skill manifest/front-matter required for discovery and invocation (name, description, argument-hint), consistent with the confirmed packaging surface from task 1.1
    - _Requirements: 1.1, 1.2_
    - _Design: Artifacts → "Files that make up the wizard skill itself"_

  - [ ] 1.3 Set up the TypeScript reference-implementation project and test tooling
    - Initialize a `tools/refimpl/` TypeScript package (tsconfig, package manifest) with a test runner and `fast-check`, plus `ajv` for JSON-schema validation
    - Add npm scripts for typecheck and single-run tests (no watch mode), and wire `shellcheck` + `BATS` invocation entrypoints for the shell-script suites
    - _Requirements: (tooling foundation for all testable units)_

- [ ] 2. Define shared domain types and the phase controller
  - [ ] 2.1 Define core domain types for the reference implementation
    - Add TypeScript types/interfaces for `PhaseId` (0–6), `ScopeChoice`, `StateFile`, `PermissionRule` (`{capability, pattern, effect}`), `HookDefinition` (versioned schema shape), `SteeringFile` (front-matter + body), and `DiffConfirmation` actions
    - Define the canonical `HookTrigger` set (the 10 Glossary triggers) and the canonical agentic-loop ordering as a single exported constant reused everywhere
    - _Requirements: 2.2, 9.1_
    - _Design: Data Models_

  - [ ] 2.2 Implement the phase controller (sequencing + jump routing + progress header)
    - Implement pure functions: `nextPhase(completed)`, `handleJump(target, scopeRecorded)`, and `renderProgressHeader(phase)` returning a header containing the phase number and total count 7
    - Enforce fixed order 0→6, route valid jumps (running Phase 1 first when scope unrecorded), reject out-of-range jumps leaving the current phase unchanged, and end after Phase 6
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7_

  - [ ]* 2.3 Write property test for phase sequencing
    - **Property 1: Phase sequencing follows the fixed order**
    - **Validates: Requirements 1.1, 1.2, 1.3, 1.7**

  - [ ]* 2.4 Write property test for jump routing
    - **Property 2: Jump routing is validated and scope-gated**
    - **Validates: Requirements 1.5, 1.6**

  - [ ]* 2.5 Write property test for progress header
    - **Property 3: Progress header identifies phase and total**
    - **Validates: Requirements 1.4**

- [ ] 3. Implement Phase 0 read-only guard and trigger enumeration
  - [ ] 3.1 Implement Phase 0 intro content model and read-only guard
    - Implement `enumerateHookTriggers()` returning exactly the canonical trigger set, and `phaseZeroFileOps()` that yields an empty mutation set (guard that blocks any create/modify/delete while Phase 0 is active)
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

  - [ ]* 3.2 Write property test for Phase 0 trigger enumeration
    - **Property 4: Phase 0 enumerates the exact Hook_Trigger set**
    - **Validates: Requirements 2.2**

  - [ ]* 3.3 Write property test for Phase 0 read-only behavior
    - **Property 5: Phase 0 performs no file mutations**
    - **Validates: Requirements 2.4**

- [ ] 4. Implement scope and path resolution
  - [ ] 4.1 Implement scope resolver, directory ensure, and under-root guard
    - Implement `resolveScope(choice)` mapping workspace→`<project>/.kiro/` and user→`~/.kiro/`, `ensureDir(path)` (idempotent create), and `isUnderRoot(path, targetRoot)` for later-phase write paths
    - Record scope id + resolved absolute target path to state before Phase 1 completes; block advance when scope unset; emit the user-scope warning
    - _Requirements: 3.1, 3.2, 3.3, 3.5, 3.6, 3.7, 3.8_

  - [ ]* 4.2 Write property test for scope resolution
    - **Property 6: Scope resolution maps to the correct .kiro root**
    - **Validates: Requirements 3.1, 3.2, 3.3**

  - [ ]* 4.3 Write property test for idempotent directory ensure
    - **Property 7: Target directory existence is ensured idempotently**
    - **Validates: Requirements 3.6**

  - [ ]* 4.4 Write property test for write-path containment
    - **Property 8: All later writes stay under the recorded scope root**
    - **Validates: Requirements 3.7**

- [ ] 5. Implement the confirm-before-write gate and the marker-scoped merge engine
  - [ ] 5.1 Implement the Diff_Confirmation write gate
    - Implement a pure state machine that emits a full-content display for creates and a diff for modifications, always offers exactly {Apply, Edit first, Skip}, writes only on Apply, loops on Edit first without writing, and leaves the file unchanged on Skip
    - Surface a write-failure-after-Apply path that leaves the target unchanged and reports the failure
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7_

  - [ ]* 5.2 Write property test for the Apply-only write condition
    - **Property 9: A file is written if and only if Apply was chosen for it**
    - **Validates: Requirements 4.3, 4.5**

  - [ ]* 5.3 Write property test for pre-write display and three-action prompt
    - **Property 10: Every write is preceded by a content/diff display offering exactly three actions**
    - **Validates: Requirements 4.1, 4.2, 4.6**

  - [ ]* 5.4 Write property test for the Edit-first loop
    - **Property 11: Edit-first loops without writing**
    - **Validates: Requirements 4.4**

  - [ ] 5.5 Implement the marker-scoped steering merge engine
    - Implement `mergeMarkers(existing, generated)`: append the generated block inside one matched `harness-builder:begin`/`:end` pair when no markers exist; replace only between markers when a pair exists; preserve every byte outside the markers
    - Add invalid-JSON guard for config targets that returns an error naming the file + parse failure and leaves the file byte-for-byte unchanged
    - _Requirements: 5.1, 5.2, 5.3, 5.5_

  - [ ]* 5.6 Write property test for content preservation outside markers
    - **Property 12: Content outside harness markers is preserved across a merge**
    - **Validates: Requirements 5.1, 5.3**

  - [ ]* 5.7 Write property test for marker-less append
    - **Property 13: Marker-less steering merge appends exactly one matched marker pair**
    - **Validates: Requirements 5.2**

- [ ] 6. Checkpoint - core control and merge foundations
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 7. Implement the permission generator (gate layer)
  - [ ] 7.1 Implement permission presets, effect resolver, and rule merge
    - Implement the four presets (deny-secrets on `fs_read`+`fs_write`, deny-destructive-shell, ask-before-network, allow-common-dev), `generateRules(selectedPresets)` (≥1 preset else error preserving selection), `resolveConflicts(rules)` (most-restrictive wins: deny>ask>allow, one effect per capability), and `mergePermissions(existing, new)` (order-preserving deduplicated union)
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 7.6, 7.7, 5.4, 5.6_

  - [ ]* 7.2 Write property test for permission merge union/idempotence
    - **Property 14: Permission merge is a deduplicated, order-preserving union and is idempotent**
    - **Validates: Requirements 5.4, 5.6**

  - [ ]* 7.3 Write property test for restrictive-effect resolution
    - **Property 15: Restrictive effect wins on capability conflicts**
    - **Validates: Requirements 7.5, 7.7**

  - [ ]* 7.4 Write property test for the deny-secrets preset
    - **Property 16: Deny-secrets preset denies both read and write on secret patterns**
    - **Validates: Requirements 7.2**

  - [ ]* 7.5 Write unit tests for preset catalog and zero-preset rejection
    - Assert each preset's capability→effect mapping and the "select at least one preset" error preserving selection state
    - _Requirements: 7.1, 7.6_

- [ ] 8. Implement the State_Manager (persistence and resume)
  - [ ] 8.1 Implement State_Manager read/record/resume logic
    - Implement `recordPhase`, `recordHookApplied`, `load()`, `computeResumeTarget(completed)` (lowest phase not completed), and `isApplied(pathOrHookId)`; rewrite the whole State_File after each completed phase and each applied hook
    - Handle write-failure (halt phase, preserve applied files), missing file (begin at Phase 1), and unreadable/unparseable file (treat progress absent, warn, retain file unchanged, begin at Phase 1)
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 6.7, 6.8_

  - [ ]* 8.2 Write property test for resume target computation
    - **Property 17: Resume target is the first uncompleted phase**
    - **Validates: Requirements 6.4**

  - [ ]* 8.3 Write property test for record-before-advance ordering
    - **Property 18: Progress is recorded before advancing**
    - **Validates: Requirements 6.1, 6.2**

  - [ ]* 8.4 Write property test for re-run idempotence
    - **Property 19: Re-running a completed phase re-applies nothing already recorded**
    - **Validates: Requirements 6.7**

  - [ ]* 8.5 Write unit tests for corrupt/missing state handling
    - Cover missing-file, unreadable, and unparseable cases and the reconfigure re-apply path
    - _Requirements: 6.5, 6.6, 6.8_

- [ ] 9. Implement the steering generator (guide layer)
  - [ ] 9.1 Implement steering section generation and manifest-evidence derivation
    - Implement `generateSection(section, scope)` producing one file per selected section with a valid `inclusion` mode (default `always`; `fileMatch` + ≥1 non-empty glob for file-type-scoped sections), an enforcement cross-reference when the section pairs with a hook/permission rule, and build/test-command guidance derived only from a command string found in a project manifest (omit + record absence otherwise)
    - Enforce skip-if-exists (do not overwrite; report skipped) and create the steering directory if absent
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 8.8, 8.9_

  - [ ]* 9.2 Write property test for one-file-per-section
    - **Property 20: Steering generation produces exactly one file per selected section**
    - **Validates: Requirements 8.2**

  - [ ]* 9.3 Write property test for valid inclusion modes
    - **Property 21: Generated steering files carry a valid inclusion mode**
    - **Validates: Requirements 8.5, 8.6**

  - [ ]* 9.4 Write property test for enforcement cross-references
    - **Property 22: Enforcement-paired steering sections cross-reference their enforcement artifact**
    - **Validates: Requirements 8.7**

  - [ ]* 9.5 Write property test for command-guidance evidence tracing
    - **Property 23: Emitted command guidance traces to real manifest evidence**
    - **Validates: Requirements 8.8**

  - [ ]* 9.6 Write unit tests for skip-existing and no-evidence paths
    - _Requirements: 8.4, 8.9_

- [ ] 10. Implement the hook generator (enforce layer) — schema mapping and trigger flow
  - [ ] 10.1 Implement the hook-schema mapper, trigger ordering, action typing, and paired deny
    - Implement `presentTriggers()` (canonical set), sequential processing in canonical order restricted to the selection, `generateHook(trigger, policy)` producing a schema-conformant `HookDefinition` stored under `<scope>/.kiro/hooks/`, `action.type` = `command` for shell / `agent` for prompt, and `proposePairedDeny(enforcementHook)`; zero-selected short-circuits (mark phase complete, advance)
    - Validate each generated definition against the versioned hook JSON schema with ajv; non-conformant or failed store → do not store, leave prior hooks unchanged, report the failure
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6, 9.7, 9.8, 9.9_

  - [ ]* 10.2 Write property test for trigger processing order and presented set
    - **Property 24: Hook trigger processing order is the canonical order restricted to the selection**
    - **Validates: Requirements 9.1, 9.2**

  - [ ]* 10.3 Write property + schema-validation test for generated hook definitions
    - **Property 25: Generated hooks are schema-conformant and correctly located** (validate every generated `.kiro/hooks/*.json` against the versioned schema via ajv)
    - **Validates: Requirements 9.5**

  - [ ]* 10.4 Write property test for action-type mapping
    - **Property 26: Action type matches the action kind**
    - **Validates: Requirements 9.7, 9.8**

  - [ ]* 10.5 Write property test for paired permission deny
    - **Property 27: Every enforcement hook proposes a paired permission deny**
    - **Validates: Requirements 9.9**

- [ ] 11. Implement natural-language policy translation and hook validators
  - [ ] 11.1 Implement NL policy validation and hook synthesis
    - Implement `generateFromNL(trigger, text)`: accept 1–2000 char mappable descriptions and present for review before any write; reject empty/>2000/unmappable with a reason and no definition; inject the `HARNESS_DISABLE_<HOOK_NAME>` kill switch (uppercased name); assemble any externally-influenced JSON via a structured serializer (`jq` typed args), never raw interpolation; resolve project paths through `${KIRO_PROJECT_DIR}` with `$PWD` fallback
    - Implement `detectConflict(hook, existingArtifacts)` (report specific artifact, withhold write, require explicit confirmation, discard on decline)
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.6, 10.7_

  - [ ]* 11.2 Write property test for NL policy validation
    - **Property 28: Natural-language policy validation accepts valid input and rejects invalid input**
    - **Validates: Requirements 10.1, 10.2**

  - [ ]* 11.3 Write property test for kill-switch naming
    - **Property 29: Generated hooks carry a correctly named kill switch**
    - **Validates: Requirements 10.3**

  - [ ]* 11.4 Write property test for structural JSON assembly
    - **Property 30: Generated command hooks assemble JSON structurally**
    - **Validates: Requirements 10.6**

  - [ ]* 11.5 Write property test for project-dir path resolution
    - **Property 31: Generated hook paths route through the project-dir variable with a cwd fallback**
    - **Validates: Requirements 10.7**

- [ ] 12. Checkpoint - generators and validators
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 13. Implement the command generator and summary reporter
  - [ ] 13.1 Implement manual-command generation
    - Implement `generateCommand(description)` producing exactly one `inclusion: manual` steering file for 1–2000 char non-empty input; reject empty/>2000 with an error and no write; refuse name collisions (no overwrite, report conflict); generate only from the user's description (no teaching examples unless requested); record written files in the State_File (retain file + leave state unchanged on record failure)
    - _Requirements: 12.1, 12.2, 12.3, 12.4, 12.5, 12.6, 12.7_

  - [ ]* 13.2 Write property test for command-description validation
    - **Property 36: Command description validation gates file generation**
    - **Validates: Requirements 12.2, 12.3**

  - [ ]* 13.3 Write property test for command file state recording
    - **Property 37: Written command files are recorded in the State_File**
    - **Validates: Requirements 12.6**

  - [ ] 13.4 Implement the summary reporter
    - Implement report assembly: one table row per applied file (path + created/merged), a "no files changed" message when empty, one verification step (executable instruction + expected observable result) per hook and per applied preset, one kill-switch entry per installed hook, per-file rollback text (restore for merged, removal for created) emitted as text and never executed, and the State_File deletion prompt (retain unless confirmed; error + retain on failed deletion)
    - _Requirements: 13.1, 13.2, 13.3, 13.4, 13.5, 13.6, 13.7_

  - [ ]* 13.5 Write property test for summary completeness
    - **Property 38: The summary report is complete over applied artifacts**
    - **Validates: Requirements 13.1, 13.3, 13.4, 13.5**

- [ ] 14. Author artifact templates and the generated-script structural guard
  - [ ] 14.1 Author the hook JSON wrapper and steering templates
    - Create `templates/hook-definition.json.tmpl` (versioned `version`/`hooks[]` schema, `command`/`agent` actions, matcher/timeout/enabled), `templates/steering/steering-section.md.tmpl` (front-matter + harness-marker block), and `templates/steering/manual-command.md.tmpl` (`inclusion: manual`)
    - _Requirements: 5.2, 8.5, 9.5, 9.7, 9.8, 12.2_

  - [ ] 14.2 Author enforcement hook script templates (fail-closed)
    - Create `templates/hooks/pretooluse-guard.sh` + `templates/hooks/pretooluse-rules.json` and `templates/hooks/stop-gate.sh`, each with the kill-switch guard, `${KIRO_PROJECT_DIR:-$PWD}` fallback, jq-only JSON, fail-closed `ask` on unusable config, and (Stop) a mandatory re-entry guard
    - _Requirements: 10.3, 10.6, 10.7, 11.2, 11.3, 11.4, 11.5_

  - [ ] 14.3 Author convenience hook script templates (fail-open)
    - Create `templates/hooks/sessionstart-context.sh`, `templates/hooks/userpromptsubmit-trigger.sh`, `templates/hooks/posttooluse-audit.sh`, and `templates/hooks/postfile-*.sh`, each with the kill switch, project-dir fallback, jq-only JSON, and non-blocking success exit within 5s on missing dependency
    - _Requirements: 10.3, 10.6, 10.7, 11.1_

  - [ ] 14.4 Implement the generated-artifact structural guard checker
    - Implement a reusable checker (used by the generator to satisfy the "refuse Stop without guard" rule and by tests) that statically verifies: kill switch present + correctly named, `${KIRO_PROJECT_DIR:-$PWD}` fallback present, JSON via jq typed args only, enforcement scripts emit `ask` (never `allow`) on unusable config, convenience scripts fail open, and `Stop` scripts contain the re-entry guard (generator refuses and errors otherwise)
    - _Requirements: 11.1, 11.2, 11.3, 11.4, 11.5, 10.3, 10.6, 10.7_

  - [ ]* 14.5 Write property tests for the structural guards over randomized policies
    - **Property 32: Convenience hooks fail open on missing dependencies** (Validates: Requirements 11.1)
    - **Property 33: Enforcement hooks fail closed on unusable configuration** (Validates: Requirements 11.2)
    - **Property 34: Stop hooks are re-entry safe and are never generated without a guard** (Validates: Requirements 11.3, 11.4)
    - **Property 35: Enforcement blocks are signaled with a bounded reason** (Validates: Requirements 11.5)

  - [ ]* 14.6 Run shellcheck and BATS over generated scripts
    - Run `shellcheck` on every template/generated script; add a BATS harness feeding crafted STDIN JSON to exercise deny/ask/allow/no-op exit paths
    - _Requirements: 11.1, 11.2, 11.3, 11.5_

- [ ] 15. Author the Kiro Skill orchestrator and reference playbooks
  - [ ] 15.1 Author `skills/wizard/SKILL.md`
    - Write the wizard contract (main-conversation only, progress header phase N of 7, confirm-before-write, merge-never-clobber), the State protocol (read state + target files at each phase start; rewrite after each phase and each applied hook), and the 7-phase dispatch table (0 Intro → 6 Summary) with jump/resume rules — consistent with the phase-controller and State_Manager behavior
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 4.2, 5.1, 6.4_

  - [ ] 15.2 Author the early-phase playbooks
    - Write `references/agentic-loop.md` (Kiro loop + guide/gate/enforce layers + the three ordering rationales + full trigger enumeration), `references/phase-scope.md`, and `references/phase-permissions.md` (presets + restrictive-wins merge)
    - _Requirements: 2.1, 2.2, 2.3, 3.1, 3.2, 3.8, 7.1, 7.7_

  - [ ] 15.3 Author the hook-phase playbooks
    - Write `references/phase-steering.md`, `references/phase-hooks.md`, and `references/hooks-catalog.md` (per-trigger firing condition, blocking capability, harness role, 2–3 canned policies, safe test, pitfalls) aligned to the confirmed Kiro trigger set
    - _Requirements: 8.1, 9.1, 9.4, 13.3_

  - [ ] 15.4 Author the generation-rules and commands playbooks
    - Write `references/artifact-rules.md` (kill switch, `KIRO_PROJECT_DIR` fallback, Stop re-entry guard, fail-open/closed split, paired permission deny, jq-only JSON) matching template task 14.x, and `references/phase-commands.md` (teach-by-example, generate manual-inclusion command from description)
    - _Requirements: 10.3, 10.6, 10.7, 11.1, 11.2, 11.4, 12.1, 12.5_

  - [ ]* 15.5 Write integration/snapshot tests for interactive and content criteria
    - Snapshot Phase 0 layer/ordering copy and full trigger enumeration; snapshot the preset catalog and per-trigger explanations; whole-file snapshots of generated steering/hook artifacts; an end-to-end resume-after-interruption walkthrough
    - _Requirements: 2.1, 2.3, 6.4, 7.1, 9.4, 13.1, 13.3_

- [ ] 16. Author repo docs and wire the skill together
  - [ ] 16.1 Author README (and README_KO parity) plus install/invoke docs
    - Document what the skill produces, the 7-phase flow, install/invocation per the confirmed packaging (task 1.1/1.2), and the kill-switch/rollback model
    - _Requirements: 1.2, 13.4, 13.5_

  - [ ] 16.2 Wire the skill and reference implementation into a consistent whole
    - Cross-check that `SKILL.md` + playbooks reference the exact template file names, trigger set, and merge/validation rules implemented in `tools/refimpl/`; run the full ajv schema validation over every template and generated fixture; ensure no orphaned template or playbook reference
    - _Requirements: 5.1, 9.5, 9.6_

- [ ] 17. Final checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional (property/unit/integration tests, shellcheck/BATS) and can be skipped for a faster MVP; core implementation and artifact-authoring tasks are never optional.
- Each task references specific requirements clauses and, where applicable, the exact design Correctness Property it validates, for traceability.
- The wizard's control logic is agent-driven Markdown; the TypeScript `tools/refimpl/` package is the testable reference substrate that the Markdown playbooks describe, so the two tracks are kept in sync by task 16.2.
- Property tests use fast-check (min 100 iterations each, one PBT per property, tagged `Feature: kiro-harness-builder, Property {number}: {property_text}`); hook definitions are validated with ajv; generated shell scripts are checked with shellcheck + BATS.
- Task 1.1 deliberately precedes any template/schema hardcoding so the Kiro hook JSON schema, permission-config location, and `KIRO_PROJECT_DIR` variable are confirmed (or their fallbacks recorded) first.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2", "1.3"] },
    { "id": 1, "tasks": ["2.1"] },
    { "id": 2, "tasks": ["2.2", "3.1", "4.1", "5.1", "5.5", "8.1"] },
    { "id": 3, "tasks": ["2.3", "2.4", "2.5", "3.2", "3.3", "4.2", "4.3", "4.4", "5.2", "5.3", "5.4", "5.6", "5.7", "8.2", "8.3", "8.4", "8.5"] },
    { "id": 4, "tasks": ["7.1", "9.1"] },
    { "id": 5, "tasks": ["7.2", "7.3", "7.4", "7.5", "9.2", "9.3", "9.4", "9.5", "9.6", "10.1"] },
    { "id": 6, "tasks": ["10.2", "10.4", "10.5", "11.1"] },
    { "id": 7, "tasks": ["11.2", "11.3", "11.4", "11.5", "13.1", "13.4", "14.1", "14.2", "14.3"] },
    { "id": 8, "tasks": ["10.3", "13.2", "13.3", "13.5", "14.4"] },
    { "id": 9, "tasks": ["14.5", "14.6", "15.1", "15.2", "15.3", "15.4"] },
    { "id": 10, "tasks": ["15.5", "16.1", "16.2"] }
  ]
}
```
