# SketchUp Ruby API quick reference (offline)

This is a compact offline guide for agents controlling SketchUp Ruby through Agent Bridge. It summarizes core API shapes and safe patterns; it is not a full replacement for the official API. The API surface is version-dependent: check `Sketchup.version` and the locally installed SketchUp version before using newer methods. Examples below use long-established APIs suitable for SketchUp 2022 unless noted.

## Runtime and model context

```ruby
Sketchup.version
model = Sketchup.active_model
model.nil?
model.entities.length
model.selection.length
```

`Sketchup.active_model` is the current model. `Model#entities` refers to the model's root entities; `Model#active_entities` refers to the current edit context (for example, inside an open group/component). Use the latter when an operation is intentionally scoped to the user's current editing context. Entities in definitions are shared by instances; modifying a component definition can affect every instance.

Common read-only inspection:

```ruby
model.entities.map(&:typename).tally
model.selection.map(&:typename)
model.entities.grep(Sketchup::Face).map { |face| [face.area, face.normal.to_a] }
```

An `Entity` may become invalid after erase/undo; check `entity.valid?` before later use. Use persistent IDs for references that must survive within a model workflow, but do not assume ordinary Ruby object references persist through reloads or model replacement.

## Model changes and undo

Group related user-visible edits in a SketchUp operation. Commit on success and abort on error; do not leave a half-finished operation open. Agent Bridge calls do not automatically create an undo transaction.

```ruby
model = Sketchup.active_model
model.start_operation('Bridge: create test geometry', true)
begin
  group = model.active_entities.add_group
  group.entities.add_face(
    [0, 0, 0], [100, 0, 0], [100, 100, 0], [0, 100, 0]
  )
  model.commit_operation
  group.entityID
rescue
  model.abort_operation
  raise
end
```

SketchUp coordinates are inches internally. Use `100.mm`, `1.m`, etc. when unit-aware lengths are needed (`Length` helpers are provided by SketchUp). Avoid saving the user's model unless explicitly requested; save/snapshot operations are also governed by Bridge file scope.

For bulk geometry construction, `EntitiesBuilder` may be faster than individual `Entities#add_*` calls, but availability/behavior depends on SketchUp version and builder constraints; verify against the linked class docs before adopting it.

## Geometry and transforms

The `Geom` module provides `Point3d`, `Vector3d`, and `Transformation`. Points are positions; vectors are directions/deltas. Ruby arrays of 3 coordinates are accepted in many APIs, but explicit objects improve readability when transforming geometry.

```ruby
origin = Geom::Point3d.new(0, 0, 0)
offset = Geom::Vector3d.new(100.mm, 0, 0)
target = origin.offset(offset)

translation = Geom::Transformation.translation(offset)
rot = Geom::Transformation.rotation(origin, Z_AXIS, 45.degrees)
combined = translation * rot
```

Transform order matters: matrix multiplication is not commutative. Confirm whether a transformation is local-to-parent or parent-to-world before applying it to nested group/component geometry. To transform an instance, prefer its `transformation`; to transform points explicitly, use `point.transform(transformation)`.

Face winding determines its front/back orientation. Inspect `face.normal`, `face.area`, and loops when geometry appears reversed or malformed. `Entities#add_face` can return `nil` when the points do not define a valid face; check the result before using it. Edges/faces added to an existing context may merge with adjacent geometry, so use groups/components to isolate generated geometry when appropriate.

## Attributes and selection

Entities support attribute dictionaries for extension-owned metadata:

```ruby
entity.set_attribute('MyExtension', 'tag', 'sample')
entity.get_attribute('MyExtension', 'tag', nil)
entity.delete_attribute('MyExtension', 'tag')
```

Use a namespaced dictionary; do not overwrite another extension's keys. Selection can be read with `model.selection`, or set/cleared through its collection API. Avoid changing selection or camera as a hidden side effect unless it is part of the requested behavior.

## Callbacks, timers, and long-running work

SketchUp UI and model APIs generally need to be invoked on SketchUp's main thread. For asynchronous work, schedule short callbacks with `UI.start_timer` and keep model/UI access on the main thread. Long calculations should not block the Bridge request indefinitely; expose observable status/result state and poll it with read-only eval calls. Moosas uses this pattern for its Main simulation and Python job handoff; see [moosas-agentbridge.md](moosas-agentbridge.md).

## Official documentation (primary sources)

These references are included so an agent can use this file without web search. The links remain useful when network access is available:

- [SketchUp Ruby API index](https://ruby.sketchup.com/)
- [Getting started with the SketchUp Ruby API](https://developer.sketchup.com/gettingstarted)
- [`Sketchup::Model`](https://ruby.sketchup.com/Sketchup/Model.html) — active/root entities, selection, operations, and model state.
- [`Sketchup::Entities`](https://ruby.sketchup.com/Sketchup/Entities.html) — adding, finding, and erasing model entities.
- [`Sketchup::EntitiesBuilder`](https://ruby.sketchup.com/Sketchup/EntitiesBuilder.html) — bulk geometry construction.
- [`Sketchup::Entity`](https://ruby.sketchup.com/Sketchup/Entity.html) — validity, IDs, attributes, and common entity behavior.
- [`Sketchup::Face`](https://ruby.sketchup.com/Sketchup/Face.html) — faces, loops, normals, area, and materials.
- [`Geom`](https://ruby.sketchup.com/Geom.html), [`Geom::Point3d`](https://ruby.sketchup.com/Geom/Point3d.html), [`Geom::Vector3d`](https://ruby.sketchup.com/Geom/Vector3d.html), and [`Geom::Transformation`](https://ruby.sketchup.com/Geom/Transformation.html) — geometry and transforms.
- [API method list](https://ruby.sketchup.com/method_list.html) — searchable method index.

