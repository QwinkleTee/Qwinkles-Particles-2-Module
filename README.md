# Part-Icles V2 — runtime module

The VFX engine from the [Part-Icles V2 plugin](https://github.com/QwinkleTee/Qwinkles-Particles-2), packaged as a ModuleScript you can require from your game code. You build effects in the plugin. You play them with this module.

## Install

Download `PartIclesModule.rbxm` from this repo and drag it into a Studio place. A ModuleScript named `Part_Icles` appears under whatever you dropped it into; move it to `ReplicatedStorage`. The source is plaintext Lua. You can read it, fork it for your own game's needs, but you can't redistribute it (see [LICENSE.md](LICENSE.md)).

## Keep the module up to date

This module is a frozen snapshot of the plugin's runtime at a specific version. When the Part-Icles 2 plugin ships an update — bug fixes, new features, performance work — the changes don't reach this module until a new `.rbxm` is pushed to this repo. To stay current:

1. Watch this repo (Watch → Releases) so GitHub pings you when a new build lands.
2. When a new build lands, download the new `PartIclesModule.rbxm`.
3. In your place, delete the existing `Part_Icles` ModuleScript under `ReplicatedStorage` and drag the new one in. Your game code doesn't change — the API is stable across updates.

Skipping updates is fine, but you'll miss whatever fixes shipped in the plugin since the version you have.

## Your first emit

You set up an effect in the plugin. You save it as a Model or Folder — usually with several Custom particles inside (a few Attachments for sparks, a Beam or two for light, maybe a PointLight for the flash). The whole thing is a self-contained tree.

At runtime, one line fires the whole tree:

```luau
local Particle = require(game.ReplicatedStorage.Part_Icles)
Particle:Activate()

Particle:AbsoluteEmit(workspace.FireballVFX)
```

`AbsoluteEmit` walks every descendant under the root you pass it and fires every emitter it finds. That includes Custom particles set up through the plugin (Parts, Attachments, Beams, Models, PointLights, Highlights, Trails) AND native Roblox emitters living alongside them — raw `ParticleEmitter` children, raw `Trail` instances, raw `Beam` instances. Each one fires per its own EmitCount, EmitDelay, EmitDuration attributes (the plugin sets these; native Roblox emitters default to 1 burst if the attributes are absent). You don't iterate the tree. You don't touch the children. You point at the root.

A typical VFX bundle mixes both kinds: a Model with two Attachments running plugin-set Custom particles, a `ParticleEmitter` directly on a child Part for a quick sparkle, and a `Trail` between two Attachments for a streak. One `AbsoluteEmit` call handles all of them with the right semantics for each kind.

That's almost every use of this module. The other recursive variants follow the same shape:

```luau
Particle:AbsoluteEnable(workspace.AuraVFX)        -- continuous rate-based emit
Particle:AbsoluteDisable(workspace.AuraVFX)       -- stop now
```

`AbsoluteEnable` is the "while X is happening" call. The player picks up the buff, you `AbsoluteEnable` their aura effect. The buff expires, you `AbsoluteDisable` it.

Timescale also has a recursive form. Slow down (or freeze, or reverse) every Custom particle and native emitter under a root for a window:

```luau
Particle:AbsoluteSetTimescale(workspace.BulletTimeVFX, 0.25, 3) -- quarter-speed for 3 seconds
Particle:AbsoluteSetTimescale(workspace.BulletTimeVFX, 0, 1)    -- frozen for 1 second
Particle:AbsoluteClearTimescale(workspace.BulletTimeVFX)        -- restore normal speed
```

You call `Activate` once at server startup (or client startup, if you're emitting from the client). You can call `AbsoluteEmit` / `AbsoluteEnable` / `AbsoluteSetTimescale` against the same tree as many times as you want.

## Emit at a custom origin (v39+)

`AbsoluteEmitAt` spawns every emitter under the root from a CFrame you supply instead of the authored position. Use it for hit-position VFX, death effects, or any "spawn this prebuilt VFX where the action happened" pattern:

```luau
Particle:AbsoluteEmitAt(workspace.ExplosionVFX, hit.Position) -- spawns at the hit point
Particle:AbsoluteEmitAt(workspace.SplashVFX, surfaceCF, {
    UseFullOrigin = true,    -- include rotation (default true)
    IgnoreLink = false,      -- respect authored Link weld (default false)
    OriginResolver = function() return livePart.CFrame end, -- live-tracking origin
})
```

How it dispatches by root kind:
- **Transformed Part-Icle root** — engine fires it with origin override threaded through `emitCtx`. The Part-Icle's 60+ attributes stay in place; no clone overhead.
- **Plain Model / BasePart container holding transformed Part-Icles AND raw emitters** — Part-Icles emit with per-node origin (`originCF * relative-offset-from-pivot`) so authored relative layout is preserved. Raw native `ParticleEmitter` / `Trail` / `Beam` get a one-time clone-move-emit-destroy pass parented to the engine's emit folder. The original stays put. Repeated `AbsoluteEmitAt` calls produce independent clones — no fight, no state sharing.

## Emit-and-wait

`AbsoluteEmit` and `AbsoluteEmitAt` return the total real-time duration in seconds across every spawned effect (EmitDelay + EmitDuration + Timescale-distorted Lifetime + PartLife, recursing nested transformed descendants). Pair with `Particle.await(n)` to yield until the last particle dies:

```luau
Particle.await(Particle:AbsoluteEmit(workspace.IntroVFX))
print("intro done")
Particle.await(Particle:AbsoluteEmitAt(workspace.MainVFX, originCF))
print("main done")
```

`.await(n)` is dot-syntax (no colon). Returns `nil` for effectively-infinite emitters (AnimateLoop without EmitDuration, Timescale-stuck-at-zero) — `Particle.await(nil)` is a no-op so the chained form stays safe.

## LinkService — standalone link tracking

`Particle.LinkService` is a runtime service that pins arbitrary parts to follow other instances using the same LinkMode dispatch (Weld / Follow / Pivot / WeldWithoutRotation) the particle engine uses internally. Useful for HUD elements, attached effects, decorative objects — anywhere you want one Part to track another without authoring a Roblox Weld constraint.

```luau
local LinkService = Particle.LinkService
LinkService:Activate()

LinkService:Link(part1, part2, "Weld")          -- infinite, full CFrame match
LinkService:Link(part1, part2, "Follow", 2.5)   -- position-only for 2.5s
LinkService:Clear(part1)                        -- stop tracking
LinkService:IsLinked(part1)                     -- boolean
LinkService:Deactivate()                        -- stop everything, restore Anchored
```

LinkService auto-anchors `part1` (and Model descendants) on `:Link` so physics doesn't fight the per-frame CFrame writes; the prior Anchored state is restored on `:Clear`. Supports `part1` types: `BasePart`, `Model`, `Attachment`. Target can be any of those plus `Bone` or `Camera`.

LinkService teardown rides along with `Particle:Deactivate()` — when the engine stops, LinkService stops too and every link restores its part's Anchored state.

## When you need finer control

The single-item methods address one Custom particle at a time. You'll reach for these less often. The shapes mirror the Absolute family:

```luau
Particle:Emit(myAttachment)                 -- one-shot on a single item
Particle:Enable(myAttachment, nil, 5)       -- continuous for 5 seconds
Particle:Disable(myAttachment)              -- stop
```

The plugin sometimes asks you to pass a per-emit parameter — a specific spawn CFrame, or an event-chain context. `EnableEmitAt` carries the CFrame:

```luau
Particle:EnableEmitAt(myAttachment, CFrame.new(0, 50, 0))  -- emit at a specific spot
```

Single-emitter timescale follows the same shape as the recursive form:

```luau
Particle:SetTimescale(myAttachment, 0.5, 2)   -- half-speed for 2 seconds
Particle:SetTimescale(myAttachment, -1, 2)    -- 2 seconds of reverse playback
Particle:ClearTimescale(myAttachment)
```

Preload before the first show:

```luau
Particle:Preload(workspace.AllVFX)   -- pin every decal texture under this root
```

## Full API

### Recursive (the common case)

| Method | What it does |
|---|---|
| `:AbsoluteEmit(root) → number?` | Walks `root` and its descendants. Fires every Custom particle plus every native `ParticleEmitter`, `Trail`, and `Beam`. Returns total wall-clock duration in seconds (nil for effectively infinite). |
| `:AbsoluteEmitAt(root, originCF, options?) → number?` | Origin-override variant. Transformed Part-Icles emit with per-node anchored origin; raw native emitters are cloned + moved + destroyed after the emit window. |
| `:AbsoluteEnable(root)` | Walks `root` and its descendants. Continuous emit on every Custom particle and native emitter. |
| `:AbsoluteDisable(root)` | Walks `root` and its descendants. Stops every Custom particle and native emitter currently running. |
| `:AbsoluteSetTimescale(root, scale, duration?, permanent?)` | Walks `root` and its descendants. Sets a timescale override on every Custom particle and native emitter. `scale=0` freezes, `<0` reverses. |
| `:AbsoluteClearTimescale(root)` | Walks `root` and its descendants. Removes timescale overrides on every Custom particle and native emitter. |

### Single-item

| Method | What it does |
|---|---|
| `:Emit(item, link?, emitCtx?)` | One-shot emit on a single Custom particle. |
| `:EmitAnimate(item, link?, emitCtx?)` | Starts the animate-mode cycle (looping visual animation, not particle spawn). |
| `:Enable(item, link?, duration?, emitCtx?)` | Continuous emit on one item. |
| `:Disable(item)` | Stops one item. Restores any animate snapshots; releases pool clones. |
| `:EnableEmit(item, link?, emitCtx?)` | Honors the `Enabled` attribute on the item's `PartIcleProperties`. |
| `:EnableEmitAt(item, cframe, options?)` | One-shot at a specific world CFrame. |
| `:SetTimescale(item, scale, duration?, permanent?)` | Override the timescale graph for a window. `scale=0` freezes, `<0` reverses. |
| `:ClearTimescale(item)` | Removes the override. |

### Engine and statics

| Method | What it does |
|---|---|
| `:Activate()` | Starts the Heartbeat loop. Call once per server or client. |
| `:Deactivate()` | Stops the engine, releases pools, disconnects listeners, tears down LinkService. |
| `:Preload(root, force?)` | Pins every decal texture under `root` so the first emit doesn't stutter. |
| `:Deload(root, force?)` | Releases pins set by `Preload`. |
| `:GetFolder()` | The folder where live particle clones get parented (under `workspace.Terrain`). |
| `:GetPoolFolder()` | The folder where idle pool clones are visible for inspection. |
| `.await(n)` | Static (dot-syntax). Yields the calling thread for `n` seconds; `nil` / `<=0` is a no-op. Pairs with `AbsoluteEmit` / `AbsoluteEmitAt` return values. |
| `.LinkService` | Standalone link-tracking sub-service. See above. |

## What the module reads from your items

Each Custom particle is a regular Roblox instance — Part, Attachment, Beam, Trail, Model, PointLight, Highlight, ImageLabel, BlurEffect, BloomEffect, ColorCorrectionEffect, or Atmosphere — that the plugin has stamped with two things: a `PartIcleProperties` Configuration child holding all the graph and range attributes (Lifetime, Rate, Color, Speed, Acceleration, Displacement, Timescale, and so on), and a `RenderTemplate` child holding the visual that gets duplicated per particle. The item itself carries a few attributes like `Transformed=true`, `EmitCount`, `EmitDelay`, `EmitDuration`, `EmissionMode`, and `AnimateLoop` — the plugin sets each one as you edit.

Optional pieces show up when you enable advanced features in the plugin. `Link` and `EmitParent` are `ObjectValue` children for parent-tracking and clone-parenting overrides. `ShapePart` points at a `BasePart` for shape-bounded emission. Folders named `MeshFlipbooks`, `BeamFlipbooks`, and `GraphBlender` hold animated-texture frames and Beam multi-stage colors.

Events are a separate subsystem. An `Events` folder inside `PartIcleProperties` holds OnEmit, OnHit, OnDeath, and OnDestruction handlers as Configuration children. Each handler points at a ModuleScript that runs when the matching event fires on a particle. Set them up through the plugin's Events panel.

You don't need to know any of this for normal use — the plugin sets the entire tree up when you click Transform. The module just expects to find what's there. If a required piece is missing (most commonly `RenderTemplate`), the emit silently no-ops for that item rather than erroring.

The module reads the V2 attribute format that the current Part-Icles 2 plugin produces. Older effects set up in earlier versions of the plugin need to be re-saved through the current plugin first.

## Why it's fast

The runtime was built to keep frame cost low even with hundreds of concurrent particles. A few things make that possible:

- **One Heartbeat connection, not one per particle.** A single `RunService.Heartbeat` listener drives every active particle in one pass. Adding or removing particles doesn't touch the connection count.
- **PreRender-phase yield for emit loops.** The continuous-emit loopBody resumes on `RunService.PreRender` (or `Heartbeat` on server) so Trail's first-frame sample reads the clean spawn pose after Roblox's attachment-transform propagation finishes. Fixes the long-standing trail-head-offset artifact on continuous emits.
- **Animation steps, not per-frame lerps.** Each particle's Lifetime is subdivided into `TotalKeyFrames` discrete steps. The Update functions step the visual state only when the particle crosses a keyframe boundary, instead of recomputing position, rotation, color, and size every frame.
- **Clone pooling.** When a particle dies, its visual clone goes back to a per-source LIFO pool instead of being destroyed. The next emit pops a clone off the pool and skips `Instance:Clone()` entirely. A TTL sweep evicts pool entries that haven't been used in a while so memory doesn't grow unbounded.
- **Zero-allocation inner loop.** The per-frame Update body for each particle reads from pre-allocated `pData` tables and writes back to the same fields. No new tables, no string concatenation, no `Instance.new` inside the hot path. Particle removal uses swap-and-pop so the active-particle array never reallocates.

The plugin's v39 release ships with `Pool` defaulting to false on newly-set-up Custom particles — flip it on per-emitter in the plugin's Advanced section when you want the throughput. The default is off because pool reuse can surface subtle visual edge cases on Beams and Trails under specific configurations; the plugin defaults conservatively, the engine itself is built for it.

## License

[LICENSE.md](LICENSE.md). Use in your games. No redistribution.

## Author

Qwinkle ([QwinkleTee on GitHub](https://github.com/QwinkleTee)). Plugin release repo: [Qwinkles-Particles-2](https://github.com/QwinkleTee/Qwinkles-Particles-2).

