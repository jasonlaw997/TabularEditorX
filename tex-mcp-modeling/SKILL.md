---
name: tex-mcp-modeling
description: Use when an agent needs to inspect, modify, optimize, or save a Tabular Editor X model, run DAX work, or manage and execute persistent user Macros through the stable Tabular Editor X MCP bridge. This skill covers instance selection, tex_model_*, tex_dax_*, tex_macro_*, and MCP self-checks; use a separate script-testing workflow for temporary C# files.
---

# TEX MCP Modeling

## Purpose

Use this skill for Tabular Editor X model, DAX, and persistent user-Macro work through the MCP bridge route only.

This skill owns:

- `tex_status`
- `tex_current_instance`
- `tex_list_instances`
- `tex_select_instance`
- `tex_mcp_self_check`
- `tex_model_*`
- `tex_dax_*`
- `tex_macro_*`

Do not use this skill to describe the local HTTP API route, `/model-tools/*`, `/dax-performance/*`, or local API port scanning.

## Executable path

1. Prefer the user-level `SKILL_TEX_PATH` environment variable when it is defined and points to the custom `TabularEditorX.exe`.
2. When working inside this repository, the expected development executable is `TabularEditor\bin\Release\TabularEditorX.exe` under the repository root.
3. Validate that the resolved file exists and is named `TabularEditorX.exe` before launching it.
4. Do not silently substitute an installed original `TabularEditor.exe` or a different repository.
5. If no valid custom executable can be resolved, ask the user for its full path.

## Bridge route

The stable MCP server key is `tex-mcp`.

Expected launch shape:

`<texExe> --mcp-bridge --auto-or-select`

This route is about:

- stdio MCP from the client;
- Tabular Editor X MCP manifest discovery;
- named-pipe instance selection;
- forwarding into the in-process Tabular Editor X backend.

## Route boundaries

- Use this skill only for MCP bridge workflows.
- Use the exposed MCP tools rather than direct HTTP requests.
- If Tabular Editor X is not running, start the custom TEX executable, wait for readiness, and then resume MCP.
- If no model is loaded, Macro list/get/create/update/delete may continue; model work, DAX execution, and Macro run require model readiness.
- Do not describe the local API route as a preferred route, backup route, or comparison route inside this skill.

## Standard flow

1. Call `tex_current_instance` when the target Tabular Editor X instance is unknown.
2. If no instance is selected, call `tex_list_instances`.
3. Select the target with `tex_select_instance`, preferably by `instanceId` or `pid`.
4. Call `tex_mcp_self_check` when backend state or feature availability is unclear.
5. Confirm the selected instance before any write.
6. Use `tex_status` when only Tabular Editor X/model readiness is needed.
7. Use `tex_model_*` for model inspection or modification.
8. Use `tex_dax_query`, `tex_dax_performance_run`, `tex_dax_performance_run_summary`, and other `tex_dax_*` tools for DAX execution and performance analysis.
9. Use `tex_macro_contexts`, `tex_macro_list`, `tex_macro_get`, `tex_macro_create`, `tex_macro_update`, and `tex_macro_delete` for persistent user-Macro management.
10. Use `tex_macro_run` only for an existing persistent user Macro while the global **Macro Tools** switch is enabled.
11. Use `tex_model_get_save_status` after model writes when save state matters.
12. Use `tex_model_save_model` only when the user explicitly asks to persist the current model source.

## Instance selection rules

- Prefer `instanceId` or `pid` for writes.
- Do not rely on `pbix` as a unique identity when multiple Tabular Editor X windows may be attached to the same PBIX.
- When multiple windows are attached to the same model, ask the user to choose the exact target if the bridge cannot unambiguously select it.
- Use `selectedInstance.summary`, `processId`, and `instanceId` when explaining the chosen target.

## Readiness rules

If the selected Tabular Editor X instance reports that no model is loaded:

- do not continue with `tex_model_*` writes;
- do not run DAX or call `tex_macro_run`;
- Macro list/get/create/update/delete may continue because they operate on the persistent user-Macro catalog;
- do not guess another transport route;
- launch or connect TEX to the intended model only when the requested operation requires it;
- resume the model-dependent operation only after readiness succeeds.

## Write safety

1. Mutating `tex_model_*` tools default to `dry_run` when `mode` is omitted.
2. Use `mode: "dry_run"` for preview, validation, and uncertain edits.
3. Use `mode: "apply"` only when the user explicitly asked to modify the current model or clearly approved the change.
4. Treat apply and save as separate actions.
5. Do not silently save the model after apply.

## Persistent user-Macro workflow

The Tabular Editor X UI exposes one **Macro Tools** switch. It synchronizes the internal management and execution feature gates. If `tex_macro_*` tools are missing, ask the user to enable **Macro Tools** in the MCP feature menu and recheck `tex_mcp_self_check` or MCP `tools/list`.

### Management

1. Use `tex_macro_contexts` before assigning `validContexts`; use only the canonical context names it returns.
2. Use `tex_macro_list` and `tex_macro_get` to obtain the current definition and `definitionHash` before changing or deleting a Macro.
3. Create, update, and delete default to `dry_run`. Use `mode: "apply"` only when the user asked to persist the operation.
4. Pass the latest `expectedDefinitionHash` to update/delete apply. Do not use `force` merely to bypass a stale definition.
5. Create/update strictly compile the complete proposed persistent user-Macro snapshot. Report `failedMacroNames`, `compilerErrors`, and readable errors when an existing invalid Macro blocks the operation.
6. These tools manage persistent user Macros in `MacroActions.json`; they do not return or modify built-in scripts.

### Execution

1. Call `tex_macro_run` with `mode: "dry_run"` first and require `executable = true` before apply.
2. Apply requires the exact latest `expectedDefinitionHash`; run never accepts `force`.
3. The global **Macro Tools** switch is the sole API/MCP execution permission. When enabled, all persistent user Macros are eligible to run; there is no per-Macro `allowExternalExecution` setting.
4. Choose the Macro selection source explicitly when the current TreeView selection is not the intended target:
   - `selectionMode: "current"` (default) passes the visible TreeView selection unchanged.
   - `selectionMode: "model"` passes the loaded model root and does not require a TreeView selection.
   - `selectionMode: "targets"` requires a non-empty `targets` array and resolves those objects without changing the visible TreeView selection. For example, use `{ "type": "Measure", "table": "Sales", "name": "Revenue" }`; table and column targets use the analogous canonical type and scope fields exposed by the tool schema.
5. The resolved selection must match the Macro's `validContexts`, and its Enabled expression must evaluate to true for that same selection. Missing, ambiguous, duplicate, or incompatible explicit targets must be corrected rather than bypassed.
6. Macro C# is not sandboxed. Enabling **Macro Tools** authorizes API/MCP callers to run any eligible persistent user Macro, so state this scope clearly when troubleshooting or explaining the switch.
7. Macro run does not automatically save the model. Save only when the user separately and explicitly requests it.

## Script-testing boundary

Do not use this skill for temporary or file-based Tabular Editor X C# script testing through `tex_script_*`. Persistent Macro management and execution remain in this skill; temporary script testing should use a separate, explicitly authorized workflow.

## Validation rules

When checking bridge state, confirm the following when present:

- `transport = tex-named-pipe-mcp`;
- `backend = in-process`;
- `modelLoaded = true` for model work;
- `selectedInstance.processId` matches the intended Tabular Editor X process;
- feature flags reflect the expected tool surface;
- when **Macro Tools** is enabled, both Macro feature flags are true and all seven `tex_macro_*` tools are exposed.

## Common tool groups

Use these groups as the mental routing guide:

- instance and health:
  - `tex_status`
  - `tex_current_instance`
  - `tex_list_instances`
  - `tex_select_instance`
  - `tex_mcp_self_check`
- model inspection and writes:
  - `tex_model_context`
  - `tex_model_list_*`
  - `tex_model_get_*`
  - mutating `tex_model_*`
- DAX execution and analysis:
  - `tex_dax_query`
  - `tex_dax_performance_run`
  - `tex_dax_performance_run_summary`
  - `tex_dax_current`
  - `tex_dax_history`
  - `tex_dax_compare*`
- persistent user-Macro management and execution:
  - `tex_macro_contexts`
  - `tex_macro_list`
  - `tex_macro_get`
  - `tex_macro_create`
  - `tex_macro_update`
  - `tex_macro_delete`
  - `tex_macro_run`
