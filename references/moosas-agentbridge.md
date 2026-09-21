# Operating Moosas through Agent Bridge

This note is derived from the checked-out Moosas SketchUp source (`skp/MoosasMain.rb` and `skp/src`). Entry points can change with the plugin; inspect the installed source before relying on details. The Bridge is generic Ruby/API access—not a Moosas-specific protocol—and no `Moosas::Commands` wrapper is required.

## What is loaded

`skp/MoosasMain.rb` requires the Ruby modules under `skp/src`, including `MoosasAnalysis`, `MoosasMainAnalysis`, `MoosasEnergy`, `MoosasWebDialog`, `MMR`, `MoosasModelPage`, and the daylight/radiance/sun-hour/ventilation modules. The Main page routes the `main_analysis` message to `MoosasAnalysis.main_analysis_async(JSON.parse(payload))`. The router (`MoosasWebDialog.receive`) first checks `MoosasUtils.moosas_active?`, which means its UI route requires a visible Moosas dialog. Direct calls to loaded Ruby methods via Bridge `eval` do not require a `Moosas::Commands` shim.

## Choose a real analysis entry point

For Main-page energy/daylight analysis, the source exposes:

```ruby
MoosasAnalysis.main_analysis_async(settings_hash)
```

It returns whether the asynchronous job was started, not the final simulation result. The payload must include valid Main-page settings. Source checks require:

- `selectBuildingType`: supported English/localized building type (Residence/居住建筑, Office/办公建筑, Hotel/酒店建筑, School/学校建筑, Commercial/商场建筑).
- `selectStandard`: a value containing `51350-2019`.
- `selectCity`: a valid key in `MoosasWeather.stations`.
- `recognize` and `radiation`: actual booleans.
- If `recognize` is false, the current active model must already have a matching, current Moosas recognition result. Geometry changes require recognition again.
- If `recognize` is true, select the intended building geometry first; recognition is then performed as part of the job.

A practical way to preserve the live UI's current valid selectors is to merge into `$ui_settings` rather than inventing station/standard values:

```ruby
settings = ($ui_settings || {}).merge('recognize' => true, 'radiation' => false)
MoosasAnalysis.main_analysis_async(settings)
```

Submit that as one narrowly scoped `eval`. Start with `recognize: true` if model recognition is needed; use `false` only when the current model is already recognized and unchanged. Set `radiation` according to the desired run and available data. This method also rejects concurrent exports/analyses and unsaved parameter drafts.

### Async results and status

The result is not returned by the initial Bridge call. The Main analysis stores internal task state (`@main_running`, `@main_status`, `@main_result`, `@main_error`) on `MoosasAnalysis`; after starting, poll with read-only Ruby expressions such as:

```ruby
{
  'running' => MoosasAnalysis.main_analysis_running?,
  'status' => MoosasAnalysis.instance_variable_get(:@main_status),
  'result' => MoosasAnalysis.instance_variable_get(:@main_result),
  'error' => MoosasAnalysis.instance_variable_get(:@main_error)
}
```

Poll at a modest interval rather than submitting another simulation. The result is also recorded through `MoosasMeta` and emitted to the UI when a visible dialog is available. A returned `true` means “started”; completion is indicated by `running: false` and a populated result or error. For failure, capture the full returned error and relevant SketchUp/Python job diagnostics without dumping private paths or unrelated user data.

## Other source-backed routes

`MoosasWebDialog.receive` routes UI actions including `daylight_analysis`, `sunhour_analysis`, `ventilation_analysis`, `radiance_analysis`, `params_analysis`, and `multi_goal_params_analysis`. These typically perform side effects and/or send results to the HTML dialog rather than returning a result to Bridge. Inspect the current handler before calling it. Some return-producing routines include `MoosasEnergy.analysis(model, building_type, require_radiation)` and `MoosasAnalysis.params_analysis_mode_1(...)`, but they have substantial prerequisites and may invoke executables or write files; prefer the supported asynchronous entry point for the full Main workflow.

The parameter sweep handlers support `wall_u`, `win_u`, `win_shgc`, and `wwr`; they send results to the dialog, and `wwr` is also used by the multi-goal route. Their inputs are arrays of hashes with `target`, `range`, `step`, `name`, and `buildingtype` as relevant. Orientation targets documented in source: 8=all, 0=south, 3=east, 1=west, 2=north, 4=roof. Treat these analyses as model-state-sensitive; inspect backup/restore behavior and current source before invoking.

## Bridge file scope and simulation prerequisites

The default Bridge policy, **Do not allow local file modifications**, is still suitable for in-memory SketchUp modeling, Ruby API inspection, and returning computed values that do not need disk/process access. It intentionally does not prevent ordinary Ruby/API control.

The checked-out Main analysis is not a pure in-memory calculation: it creates job directories and request/result/progress/log files under the Moosas data area, copies recognized RDF and weather inputs, and starts the bundled Python runtime. Thus the default no-file-modification scope can block the genuine end-to-end simulation. If the user requests that simulation, explain the concrete requirement and use the Bridge's **specified directory** scope for the relevant Moosas data/work area when sufficient; **all directories** is the broad ⚠️ option and should be used only when explicitly requested. Scope also governs Bridge `load_file`, `reload`, and `save_snapshot`. Do not try to bypass Bridge guards with a different filesystem or process API.

These are in-process guards, not a security boundary. The Bridge exposes Ruby `eval` by design; never describe no-write mode as an OS-level sandbox. Return only the requested numerical/semantic result—do not expose model or job paths.

## Current source landmarks

- `skp/MoosasMain.rb`: extension load order and initialization.
- `skp/src/MoosasWebDialog.rb`: UI message router and visibility-gated `receive`.
- `skp/src/MoosasMainAnalysis.rb`: Main analysis validation, async scheduling, job files, Python handoff, result/error storage.
- `skp/src/MoosasAnalysis.rb`: parameter sweeps and energy analysis orchestration.
- `skp/src/MoosasEnergy.rb`: synchronous energy method and its prerequisites/external simulator.

