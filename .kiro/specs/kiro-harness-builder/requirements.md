# Requirements Document

## Introduction

`kiro-harness-builder` is an interactive wizard that builds a project "harness" out of **Kiro's native features** — steering, capability-based permissions, and agent hooks — rather than out of prompt text alone. It adapts the existing `claude-harness-builder` concept from Claude Code's mechanisms to Kiro's mechanisms.

The core philosophy is unchanged: guardrails should be enforced by **structure, not by hope**. A rule written only into a steering file expresses an intention the model may forget as context grows; nothing enforces it. Kiro provides three native layers that the wizard composes, in a deliberate order:

- **Steering (guide)** — persistent Markdown context (`.kiro/steering/*.md`) with `always`, `fileMatch`, and `manual` inclusion modes. Manual steering files also act as on-demand slash commands.
- **Permissions (gate)** — Kiro's capability-based permission rules (`deny` / `ask` / `allow`) that the agent runtime evaluates independently of the model, where restrictive rules win and cannot be loosened.
- **Hooks (enforce)** — deterministic actions bound to session and tool events (`.kiro/hooks/*.json`) that can inject context, block a tool call, or gate the end of a turn.

The wizard walks the user through these layers in a confirm-before-write flow, turns natural-language policies into real hook and permission artifacts, and applies every write only after showing a diff. It is resumable so an interrupted run can continue safely.

> **Assumptions requiring confirmation** (see "Open Questions for Review" at the end): the exact Kiro hook event set to cover, how the wizard itself is packaged/distributed in Kiro, and the precise on-disk location/format of Kiro's permission configuration on Kiro Web. Requirements below encode reasonable defaults for these; they will be refined based on user feedback.

## Glossary

- **Harness**: The collection of Kiro-native artifacts (steering files, permission rules, hooks) installed into a target project to guide, gate, and enforce agent behavior.
- **Harness_Builder**: The overall interactive wizard system that produces the Harness. Umbrella term for the components below.
- **Scope_Selector**: The Harness_Builder component that determines where harness artifacts are written (workspace level vs. user level).
- **Permission_Generator**: The Harness_Builder component that produces Kiro capability-based permission rules.
- **Steering_Generator**: The Harness_Builder component that produces Kiro steering files.
- **Hook_Generator**: The Harness_Builder component that produces Kiro agent hook definitions.
- **Command_Generator**: The Harness_Builder component that produces manual-inclusion steering files that act as slash commands.
- **Summary_Reporter**: The Harness_Builder component that reports applied files, verification steps, kill switches, and rollback instructions.
- **State_Manager**: The Harness_Builder component that persists and restores wizard progress for resume-after-interruption.
- **Capability**: A Kiro permission target — one of `fs_read`, `fs_write`, `shell`, `web_fetch`, `web_search`, `mcp`, `subagent`, `skill`, `power`, `context`, `diagnostics`, `sandbox_network`, or a meta-capability (`all`, `builtin`, `filesystem`).
- **Permission_Rule**: A declarative rule pairing a Capability and match pattern with an effect (`deny`, `ask`, or `allow`).
- **Inclusion_Mode**: A steering file's activation setting — `always`, `fileMatch` (glob-triggered), or `manual` (on-demand / slash command).
- **Hook_Definition**: A single Kiro hook entry (name, trigger, optional matcher, action) stored within a hook file under `.kiro/hooks/`.
- **Hook_Trigger**: The event that fires a hook — one of `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Stop`, `PreTaskExec`, `PostTaskExec`, `PostFileCreate`, `PostFileSave`, `PostFileDelete`.
- **Enforcement_Hook**: A hook whose purpose is to block or gate (a `PreToolUse`, `UserPromptSubmit`, or `Stop` hook that can reject an action).
- **Convenience_Hook**: A hook whose purpose is context injection, auditing, or notification, which never blocks work.
- **Kill_Switch**: A per-hook environment-variable escape hatch (named `HARNESS_DISABLE_<HOOK_NAME>`) that disables an individual generated hook.
- **State_File**: The JSON file (`.harness-builder-state.json`) recording completed phases and applied files for resume.
- **Wizard_Phase**: One numbered step of the wizard (0 Intro, 1 Scope, 2 Permissions, 3 Steering, 4 Hooks, 5 Commands, 6 Summary).
- **Diff_Confirmation**: A single-select approval prompt (Apply / Edit first / Skip) shown before any file write.

## Requirements

### Requirement 1: Guided multi-phase wizard flow

**User Story:** As a developer, I want the harness to be built through an ordered, guided wizard, so that I understand each native layer and apply it in the correct dependency order.

#### Acceptance Criteria

1. WHEN the Harness_Builder starts a new run, THE Harness_Builder SHALL present Wizard_Phase 0 (Intro) before any other phase.
2. THE Harness_Builder SHALL sequence the 7 Wizard_Phases in the fixed order: 0 Intro, 1 Scope, 2 Permissions, 3 Steering, 4 Hooks, 5 Commands, 6 Summary.
3. WHEN the user confirms completion of a non-final Wizard_Phase (0 through 5), where completion means all required inputs defined for that phase have been recorded, THE Harness_Builder SHALL present the next Wizard_Phase in the fixed order.
4. WHILE presenting a Wizard_Phase, THE Harness_Builder SHALL display a progress indicator identifying the current phase number (0 to 6) and the total phase count (7).
5. IF the user requests to jump to a specific Wizard_Phase in the range 0 to 6 before Scope has been recorded, THEN THE Harness_Builder SHALL run Wizard_Phase 1 (Scope) before the requested phase.
6. IF the user requests to jump to a Wizard_Phase outside the range 0 to 6, THEN THE Harness_Builder SHALL reject the request, display an error message indicating the requested phase is invalid, and remain on the current Wizard_Phase.
7. WHEN the user confirms completion of Wizard_Phase 6 (Summary), THE Harness_Builder SHALL end the wizard run and present no further Wizard_Phase.

### Requirement 2: Introduce Kiro-native intervention points

**User Story:** As a developer, I want an overview of Kiro's agentic loop and where each layer intervenes, so that I can make informed choices in later phases.

#### Acceptance Criteria

1. WHEN Wizard_Phase 0 is presented, THE Harness_Builder SHALL present all three native layers—Steering, Permissions, and Hooks—where each layer includes its name, its one-word role designation (Steering = guide, Permissions = gate, Hooks = enforce), and at least one sentence describing its function.
2. WHEN Wizard_Phase 0 is presented, THE Harness_Builder SHALL enumerate every Hook_Trigger event defined in the Glossary at which the Harness can intervene in the agentic loop, presenting each event by name with no omissions from the defined set.
3. WHEN Wizard_Phase 0 is presented, THE Harness_Builder SHALL explain the rationale for the phase ordering by stating each of the three ordering relationships—scope precedes permissions, permissions precede hooks, and steering precedes hooks—with a reason for each relationship.
4. WHILE Wizard_Phase 0 is active, THE Harness_Builder SHALL complete Wizard_Phase 0 without creating, modifying, or deleting any file in the workspace.
5. IF a file creation, modification, or deletion operation is attempted during Wizard_Phase 0, THEN THE Harness_Builder SHALL block the operation, leave all workspace files unchanged, and present a message indicating that Wizard_Phase 0 is read-only.

### Requirement 3: Select installation scope

**User Story:** As a developer, I want to choose where the harness is installed, so that it applies at the right level for my team or machine.

#### Acceptance Criteria

1. WHEN Wizard_Phase 1 is presented, THE Scope_Selector SHALL offer a workspace-level scope option that resolves its write target to the project `.kiro/` directory.
2. WHEN Wizard_Phase 1 is presented, THE Scope_Selector SHALL offer a user-level scope option that resolves its write target to the user `~/.kiro/` directory.
3. WHEN the user selects a scope, THE Scope_Selector SHALL record the selected scope identifier and its resolved absolute target path in the State_File before Wizard_Phase 1 completes.
4. IF writing the selected scope to the State_File fails, THEN THE Scope_Selector SHALL leave the State_File unchanged from its pre-write contents, present an error indication that the scope could not be saved, and remain on Wizard_Phase 1.
5. IF the user attempts to advance from Wizard_Phase 1 without selecting a scope, THEN THE Scope_Selector SHALL block advancement to Wizard_Phase 2 and present an indication that a scope selection is required.
6. IF the resolved target directory for the selected scope does not exist, THEN THE Scope_Selector SHALL create the target directory before recording its resolved target path in the State_File.
7. THE Scope_Selector SHALL apply the scope recorded in the State_File as the write target for Wizard_Phases 2 through 5.
8. WHERE the user selects the user-level scope, THE Scope_Selector SHALL present a warning that hooks referencing workspace-relative paths do not apply to other projects.

### Requirement 4: Confirm before every write

**User Story:** As a developer, I want to review exact content before anything is written, so that the wizard never changes my project without my approval.

#### Acceptance Criteria

1. WHEN the Harness_Builder is about to create a new file, THE Harness_Builder SHALL display the complete proposed file content before writing.
2. WHEN proposed content or a diff is displayed, THE Harness_Builder SHALL present a Diff_Confirmation offering exactly three selectable actions: Apply, Edit first, and Skip.
3. IF the user selects Skip, THEN THE Harness_Builder SHALL leave the target file unchanged and continue to the next step.
4. IF the user selects Edit first, THEN THE Harness_Builder SHALL incorporate the user's requested changes, present an updated Diff_Confirmation with the same Apply, Edit first, and Skip actions, and repeat this cycle each time Edit first is selected without writing the file.
5. THE Harness_Builder SHALL write a file only after the user selects Apply for that file, and SHALL NOT modify any file for which the user has not selected Apply.
6. WHEN the Harness_Builder is about to modify an existing file, THE Harness_Builder SHALL display a diff of the proposed changes against the current file content before writing.
7. IF a file write operation fails after the user selects Apply, THEN THE Harness_Builder SHALL leave the target file unchanged, present an error indication that the write did not complete, and continue to the next step.

### Requirement 5: Merge into existing files without clobbering

**User Story:** As a developer with existing Kiro configuration, I want the wizard to preserve my current settings, so that installing the harness never discards my work.

#### Acceptance Criteria

1. WHEN a target file already exists, THE Harness_Builder SHALL write new content such that every byte of pre-existing content outside the harness-builder marker boundaries remains identical to its pre-merge state.
2. WHEN merging steering content into an existing steering file that contains no harness-builder markers, THE Steering_Generator SHALL append the generated content enclosed within a matched pair of begin and end harness-builder marker lines.
3. WHEN merging steering content into an existing steering file that already contains a matched pair of harness-builder markers, THE Steering_Generator SHALL replace only the content between those markers and SHALL leave all content outside the markers unchanged.
4. WHEN merging Permission_Rules into an existing permission configuration, THE Permission_Generator SHALL write the set union of existing and new rules, retaining all existing rules and adding only rules not already present, with no duplicate rule entries.
5. IF a target configuration file contains content that cannot be parsed as valid JSON, THEN THE Harness_Builder SHALL leave the file byte-for-byte unchanged AND SHALL return an error indication identifying the file and the parse failure.
6. WHEN a generated artifact identical to one already present in a target file is being merged, THE Harness_Builder SHALL make no change to that target file for that artifact.

### Requirement 6: Persist and resume wizard progress

**User Story:** As a developer, I want the wizard to remember progress, so that an interrupted or compacted session can resume without losing work or repeating writes.

#### Acceptance Criteria

1. WHEN a Wizard_Phase completes, THE State_Manager SHALL record in the State_File the completed phase identifier and the list of file paths applied during that phase before the next phase begins.
2. WHEN an individual hook is applied within Wizard_Phase 4, THE State_Manager SHALL update the State_File to record that hook's identifier as applied before applying the next hook.
3. IF a write to the State_File fails, THEN THE State_Manager SHALL halt the current phase, display an error message indicating that progress could not be saved, and SHALL preserve any files already applied in the current session.
4. WHEN the Harness_Builder is invoked and a readable, valid State_File exists, THE Harness_Builder SHALL prompt the user to either resume at the first phase not recorded as completed or restart from Wizard_Phase 1.
5. WHEN the Harness_Builder is invoked and no State_File exists, THE Harness_Builder SHALL begin at Wizard_Phase 1.
6. IF the State_File is unreadable or cannot be parsed as valid state, THEN THE Harness_Builder SHALL treat recorded progress as absent, display a warning message indicating that saved progress could not be read, retain the existing State_File unchanged, and begin at Wizard_Phase 1.
7. WHEN the user re-runs a phase recorded as completed, THE Harness_Builder SHALL skip each artifact whose file path or hook identifier is already recorded as applied in the State_File.
8. WHERE the user chooses to reconfigure a specific recorded artifact, THE Harness_Builder SHALL re-apply that artifact and update its record in the State_File.

### Requirement 7: Generate capability-based permission rules (gate layer)

**User Story:** As a developer, I want a permission wall built from presets, so that dangerous capabilities are denied or gated independently of the model's cooperation.

#### Acceptance Criteria

1. WHEN Wizard_Phase 2 is presented, THE Permission_Generator SHALL display all available presets, where each preset lists the Kiro Capabilities it maps to and the effect (`deny`, `ask`, or `allow`) applied to each capability.
2. THE Permission_Generator SHALL offer a preset that assigns the `deny` effect to both read and write access for files identified as secret or credential files, applied through `fs_read` and `fs_write` deny rules.
3. THE Permission_Generator SHALL offer a preset that assigns the `deny` effect to `shell` operations classified as destructive, where destructive is defined as operations that delete, overwrite, or relocate files or that modify system state outside the project workspace.
4. THE Permission_Generator SHALL offer a preset that assigns the `ask` effect to the network capabilities `web_fetch` and `sandbox_network`.
5. WHEN the user selects one or more presets, THE Permission_Generator SHALL generate Permission_Rules that assign each affected Kiro Capability exactly one effect from the set `deny`, `ask`, and `allow`.
6. IF the user attempts to generate Permission_Rules with zero presets selected, THEN THE Permission_Generator SHALL NOT produce Permission_Rules and SHALL return an indication that at least one preset must be selected, preserving the current selection state.
7. WHERE two or more selected Permission_Rules assign different effects to the same Kiro Capability, THE Permission_Generator SHALL apply the single most restrictive effect, ordering `deny` as more restrictive than `ask` and `ask` as more restrictive than `allow`.

### Requirement 8: Generate steering files (guide layer)

**User Story:** As a developer, I want a project rules skeleton as steering, so that Kiro consistently follows my conventions across sessions.

#### Acceptance Criteria

1. WHEN Wizard_Phase 3 is presented, THE Steering_Generator SHALL display a list of at least 1 selectable steering section, where each list entry shows the section name and a one-line description of its scope.
2. WHEN the user confirms a selection of 1 or more steering sections, THE Steering_Generator SHALL generate exactly one steering file per selected section under the recorded scope's `steering` directory.
3. IF the recorded scope's `steering` directory does not exist when generation begins, THEN THE Steering_Generator SHALL create the directory before writing steering files.
4. IF a target steering file already exists at the resolved path, THEN THE Steering_Generator SHALL not overwrite it, SHALL retain the existing file unchanged, and SHALL return an indication that the file was skipped.
5. WHEN generating a steering file, THE Steering_Generator SHALL set the file's Inclusion_Mode to exactly one of `always`, `fileMatch`, or `manual`, defaulting to `always` when the section does not specify a mode.
6. WHERE a steering section applies only to specific file types, THE Steering_Generator SHALL set that steering file's Inclusion_Mode to `fileMatch` and SHALL record at least one non-empty glob pattern for that mode.
7. WHEN a generated steering section corresponds to an Enforcement_Hook or Permission_Rule, THE Steering_Generator SHALL include a cross-reference identifying the paired enforcement artifact by its name or path.
8. WHEN generating build-command or test-command guidance, THE Steering_Generator SHALL derive each referenced command from a command string found in a configuration or manifest file within the target project.
9. IF no build-command or test-command evidence is found in the target project, THEN THE Steering_Generator SHALL omit the corresponding command guidance and SHALL record an indication that no supporting evidence was found.

### Requirement 9: Select and generate agent hooks (enforce layer)

**User Story:** As a developer, I want to choose which loop events to guard and generate hooks for them, so that behavior is enforced deterministically at fixed events.

#### Acceptance Criteria

1. WHEN Wizard_Phase 4 begins, THE Hook_Generator SHALL present the supported Hook_Trigger events (as defined in the Glossary) as a multi-select list.
2. WHEN the user selects one or more Hook_Trigger events, THE Hook_Generator SHALL process the selected events sequentially, one event at a time, in agentic-loop order.
3. IF the user selects zero Hook_Trigger events, THEN THE Hook_Generator SHALL skip hook generation, mark Wizard_Phase 4 complete, and advance to the next Wizard phase.
4. WHEN processing a selected Hook_Trigger, THE Hook_Generator SHALL present that event's firing condition, its blocking capability, and its harness role, and SHALL offer the 2 to 3 canned Policy options defined for that event.
5. WHEN the user chooses a Policy for a Hook_Trigger, THE Hook_Generator SHALL generate a Hook_Definition that conforms to the versioned hook JSON schema and SHALL store it under the scope's `.kiro/hooks/` directory.
6. IF generating or storing a Hook_Definition fails, or the generated Hook_Definition does not conform to the versioned hook JSON schema, THEN THE Hook_Generator SHALL not store that Hook_Definition, SHALL leave all previously stored Hook_Definitions unchanged, and SHALL present an error indication describing the failure.
7. WHEN a generated hook's action is a shell command, THE Hook_Generator SHALL set the Hook_Definition action type to `command`.
8. WHEN a generated hook's action is an agent prompt, THE Hook_Generator SHALL set the Hook_Definition action type to `agent`.
9. WHERE an Enforcement_Hook is generated, THE Hook_Generator SHALL propose a paired Permission_Rule that enforces the same Policy at the gate layer.

### Requirement 10: Translate natural-language policies into hook artifacts

**User Story:** As a developer, I want to describe a policy in plain language and get a working hook, so that I do not have to author hook scripts by hand.

#### Acceptance Criteria

1. WHEN the user submits a natural-language policy description of 1 to 2000 characters for a selected Hook_Trigger, THE Hook_Generator SHALL generate a Hook_Definition implementing that policy and present it to the user for review before it is written to disk.
2. IF the submitted natural-language policy description is empty, exceeds 2000 characters, or does not map to any supported Hook_Trigger, THEN THE Hook_Generator SHALL reject the request, produce an error indication stating the reason, and generate no Hook_Definition.
3. WHEN generating a hook from a natural-language policy, THE Hook_Generator SHALL include a Kill_Switch guard named `HARNESS_DISABLE_<HOOK_NAME>`, where `<HOOK_NAME>` is the uppercased generated hook name, that causes the hook to exit without performing its action when the variable is set to a non-empty value.
4. IF a described policy contradicts a previously generated Permission_Rule or Enforcement_Hook, THEN THE Hook_Generator SHALL report the specific conflicting artifact, withhold applying the new Hook_Definition, and require an explicit user confirmation before applying it.
5. IF the user declines the confirmation requested for a reported conflict, THEN THE Hook_Generator SHALL discard the generated Hook_Definition and leave all previously generated artifacts unchanged.
6. WHEN generating a command-action hook that assembles JSON from externally influenced input, THE Hook_Generator SHALL construct the JSON using a structured serialization tool and SHALL NOT build the JSON through raw string interpolation of that input.
7. WHEN generating a hook that references project paths, THE Hook_Generator SHALL resolve each project path through a project-directory variable, and IF that variable is unset or empty, THEN THE Hook_Generator SHALL resolve the path relative to the current working directory.

### Requirement 11: Apply safe failure behavior to generated hooks

**User Story:** As a developer, I want generated hooks to fail safely, so that a broken hook neither traps my session nor silently disables a safeguard.

#### Acceptance Criteria

1. WHERE a generated hook is a Convenience_Hook, IF any dependency required by the hook is unavailable, THEN THE generated hook SHALL terminate using the runtime's documented non-blocking (success) exit signal within 5 seconds and SHALL allow the triggering action to proceed unchanged.
2. WHERE a generated hook is an Enforcement_Hook, IF its policy configuration cannot be parsed or evaluated for any reason, THEN THE generated hook SHALL emit an `ask` decision accompanied by a non-empty reason string (1 to 500 characters) indicating the evaluation failure, and SHALL NOT emit an `allow` decision.
3. WHEN a generated `Stop` hook is invoked for a turn it has already gated, THE generated hook SHALL detect the prior gating via its re-entry guard and SHALL terminate using the runtime's documented non-blocking (success) exit signal without gating the turn a second time.
4. WHEN THE Hook_Generator is requested to generate a `Stop` hook, IF the hook definition lacks a re-entry guard, THEN THE Hook_Generator SHALL refuse to produce the hook and SHALL return an error indication identifying the missing re-entry guard, and SHALL NOT write any hook artifact.
5. WHEN a generated Enforcement_Hook blocks an action, THE generated hook SHALL signal the block using the runtime's documented blocking exit signal or blocking decision output, and SHALL include a non-empty reason string (1 to 500 characters) describing the block.

### Requirement 12: Generate manual steering slash commands

**User Story:** As a developer, I want reusable commands generated from my description, so that I can invoke repeatable procedures on demand.

#### Acceptance Criteria

1. WHEN Wizard_Phase 5 is presented, THE Command_Generator SHALL display an explanation stating that manual-inclusion steering files are invoked on demand as slash commands and are not loaded automatically.
2. WHEN the user submits a non-empty command description containing between 1 and 2000 characters, THE Command_Generator SHALL generate one steering file with Inclusion_Mode set to `manual` under the scope's `steering` directory.
3. IF the user submits a command description that is empty or exceeds 2000 characters, THEN THE Command_Generator SHALL reject the request, SHALL NOT create any steering file, and SHALL present an error message indicating that the description is empty or too long.
4. IF a generated command's target file name already exists in the scope's `steering` directory, THEN THE Command_Generator SHALL NOT overwrite the existing file and SHALL present an error message indicating a naming conflict.
5. THE Command_Generator SHALL generate a command only from the user's own submitted description and SHALL NOT install teaching-example commands unless the user explicitly requests them.
6. WHEN a generated command file is successfully written, THE Command_Generator SHALL record it in the State_File as an applied file.
7. IF recording an applied file in the State_File fails, THEN THE Command_Generator SHALL retain the generated command file, SHALL present an error message indicating the state-recording failure, and SHALL leave the State_File unchanged from its prior valid content.

### Requirement 13: Report verification, kill switches, and rollback

**User Story:** As a developer, I want a closing summary, so that I can verify the harness works, disable pieces of it, and roll back changes.

#### Acceptance Criteria

1. WHEN Wizard_Phase 6 is presented, THE Summary_Reporter SHALL display a table listing every file created or modified during the run, where each entry includes the file path and its operation type (created or merged) as recorded in the State_File.
2. IF no files were created or modified during the run, THEN THE Summary_Reporter SHALL display a message indicating that no files were changed.
3. WHEN Wizard_Phase 6 is presented, THE Summary_Reporter SHALL provide, for each generated hook and each applied permission preset, a verification step consisting of an executable instruction the developer can run and the expected observable result that confirms correct installation.
4. WHEN Wizard_Phase 6 is presented, THE Summary_Reporter SHALL list every Kill_Switch installed during the run, where each entry identifies the switch and the action required to disable the corresponding hook or all hooks.
5. WHEN Wizard_Phase 6 is presented, THE Summary_Reporter SHALL provide rollback instructions for each created or merged file, providing a restore instruction for each merged file and a removal instruction for each created file, and SHALL display removal instructions as text without executing them.
6. WHEN Wizard_Phase 6 completes, THE Summary_Reporter SHALL prompt the user to confirm deletion of the State_File, and SHALL retain the State_File unless the user explicitly confirms deletion.
7. IF the user confirms deletion of the State_File and the deletion does not succeed, THEN THE Summary_Reporter SHALL display an error indication that deletion failed and SHALL retain the State_File.
