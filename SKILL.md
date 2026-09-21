---
name: sketchup-agent
description: Control and debug SketchUp Ruby through the local Agent Bridge in an already-open SketchUp session, including extension workflows and Moosas simulations. Use for SketchUp Ruby/API debugging; not generic UI automation or unrelated Ruby work.
---

# SketchUp Agent Bridge

Use the local Agent Bridge to control the Ruby API in the user's already-open SketchUp session. The success criterion is working Ruby API control; an extension does not need a `Moosas::Commands` namespace or any other prescribed command wrapper.

## References

- For Moosas analysis/simulation entry points, prerequisites, asynchronous results, and Bridge scope implications, read [references/moosas-agentbridge.md](references/moosas-agentbridge.md).
- For offline SketchUp Ruby API guidance on model/entities, geometry, transformations, and safe model operations, read [references/sketchup-ruby-api.md](references/sketchup-ruby-api.md).

## Agent Bridge behavior and scope

- Work through the authenticated Bridge in the existing SketchUp session. Do not start SketchUp, use mouse automation, or switch to another session.
- Ruby `eval` is always enabled; there is no separate "Allow raw eval" switch. Use narrowly scoped expressions to call Ruby APIs, inspect state, and return useful computation results.
- File access has three policies: **Do not allow local file modifications** (default), **Allow modifications in a specified directory**, and **Allow modifications in all directories** (broad access; normally avoid). The selected policy also constrains Bridge `load_file`, `reload`, and `save_snapshot` operations. File/process/model-save guards are defense in depth, not an OS sandbox; arbitrary in-process Ruby or native extensions cannot be guaranteed incapable of external writes.
- With file modifications disallowed, Ruby API access and in-memory SketchUp modeling remain available; avoid filesystem-dependent workflows and never imply that paths are exposed. Some real analyses need files or external processes and therefore require a suitable explicitly selected scope.
- Do not return model file paths, connection-file contents, or Bridge tokens. The client reads `%APPDATA%\\AgentBridge\\bridge_connection.json` (with platform temp-directory fallbacks); never print or persist its token.

## Connection and calls

Use the project-provided `bridge_client.py` when available. Before changing a model, verify the Bridge and intended model:

```powershell
python .\bridge_client.py ping
python .\bridge_client.py model_info
```

Typical operations:

```powershell
python .\bridge_client.py eval "Sketchup.active_model.entities.length"
python .\bridge_client.py eval "Sketchup.active_model.entities.add_group.name = 'Bridge test'"
python .\bridge_client.py load_file "D:/project/extension/tool.rb"
python .\bridge_client.py reload "[\"D:/project/extension/dependency.rb\", \"D:/project/extension/tool.rb\"]"
python .\bridge_client.py run ui_open --ruby "MoosasWebDialog.show_ui"
```

`run <alias>` uses a configured command map. Otherwise `run <name> --ruby <qualified-no-argument-call>` can invoke any loaded Ruby/API method; use `eval` for parameterized expressions and values. Treat `ok: false` as failure and inspect the full structured backtrace before editing source. A Bridge timeout may mean a modal dialog or a long-running task; check SketchUp before issuing another model mutation.

## Installing the bundled RBZ

This skill package includes `Agent_Bridge.rbz`, the installation package for the Agent Bridge extension. An RBZ is a ZIP archive: its root files and folders must be installed directly in SketchUp's `Plugins` directory. Do not leave the RBZ itself in that directory and do not extract it into an extra `Agent_Bridge.rbz` or `Agent_Bridge` parent folder.

When the user asks to install or update the bundled extension, the agent may perform the installation without UI automation:

1. Inspect the RBZ first and verify that it contains its root registration file and matching support folder. For the bundled package these are `Agent_Bridge.rb` and `Agent_Bridge/`.
2. Use the active SketchUp session to obtain its actual `Plugins` directory. Confirm that the directory belongs to the intended SketchUp installation.
3. If a matching installed extension exists, preserve it as a dated backup before replacing only `Agent_Bridge.rb` and `Agent_Bridge/`. Never overwrite unrelated extensions.
4. Extract the two verified root items into `Plugins`, preserving their relative paths. This is equivalent to Extension Manager's RBZ installation; no separate manual registration step is required because the root RB file calls `Sketchup.register_extension`.
5. Restart SketchUp to load an updated extension cleanly. Do not attempt to load a second copy into an already-loaded Ruby process. After restart, verify with `ping` and `model_info`; if the extension is disabled, enable it in Extension Manager.

This workflow changes the SketchUp installation, so use it only for the RBZ bundled with this skill or an RBZ explicitly supplied and authorized by the user. Do not install extensions downloaded from arbitrary sources. Keep Bridge tokens and connection-file contents private throughout installation and verification.

## Debugging and mutation discipline

Inspect the actual loaded code and current diff before editing. Use the extension's real entry point where available, but do not require a particular wrapper. Business code owns SketchUp operation boundaries (`start_operation`, `commit_operation`, `abort_operation`); the Bridge must not wrap arbitrary calls in an operation. Reload changed Ruby files in dependency order and re-run the same check. Preserve user changes and recover failed model mutations with the public undo operation or a clean fixture, not stacked partial retries.

For model-mutating tests, record relevant pre/post model state and semantic results. If a task requires clean-session validation, close and reopen SketchUp on a fresh runtime copy of the fixture and repeat the check; do not replace the user's open session or model.
