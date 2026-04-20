# Phase 2 Report: Pipeline & API Isolation

## Completion Status: DONE

## Changes Made

### 1. PostFxPipeline Infrastructure (Level 3 - Pipeline Deconstruction)
- **Problem**: `Renderer.applyPostFX` was wrapped 4 times in an onion model:
  - Layer 1 (experience_optimized): grain, vignette, depth fog
  - Layer 2 (weather tint): color overlay from weather - **REDUNDANT**
  - Layer 3 (optimized weather tint): supersedes Layer 2 by zeroing its params
  - Layer 4 (underwater fog): depth-based water overlay
  
  Each wrapper called `prev.call(this, ...)`, creating a 4-deep prototype chain lookup that broke V8 JIT optimization. The wrappers also fought each other - Layer 3 explicitly suppressed Layer 2 by zeroing its parameters before the call.

- **Fix**: Introduced `TU.PostFxPipeline` - a linear, priority-ordered pipeline where stages register by name and priority number.
  - API: `register(name, priority, fn)`, `execute(renderer, time, depth01, reducedMotion)`, `list()`, `remove(name)`
  - Pipeline order: Base(10) -> WeatherTint(20) -> UnderwaterFog(30) -> Lightning(40) -> Complete(99)
  - Existing patches continue to work (backward compatible) while new code can use the pipeline.

### 2. Eliminated Redundant Weather Tint Wrapper (Layer 2)
- **Problem**: `__weatherPostTintInstalled` (Layer 2) installed a weather color tinting wrapper, but `__weatherPostTintOptimized` (Layer 3) completely superseded it by:
  1. Zeroing the weatherFx params (`fx.postA = 0, fx.lightning = 0`)
  2. Calling Layer 2 (which now does nothing due to zeroed params)
  3. Restoring params and applying its own optimized tint
  
  This means Layer 2 was a no-op that added call overhead on every frame.

- **Fix**: Pre-set `Renderer.prototype.__weatherPostTintInstalled = true` so Layer 2 never installs. Layer 3 now wraps Layer 1 directly, removing one function call from the hot rendering path.

### 3. Centralized Weather State (Single Source of Truth)
- **Problem**: Weather state was scattered across `game.weather`, `window.AppServices.get('weatherFx')`, and various local variables in different patches. Different patches wrote to `fx.postR`, `fx.postG`, etc. independently.
- **Fix**: Created `TU._weatherState` as the canonical weather state object with:
  - Weather type/intensity/target
  - PostFX color tint parameters
  - Acid rain flag
  - Registered the `fx` sub-object as the `weatherFx` service

### 4. Canvas getContext Hijack (Already Fixed in Phase 0)
- The global `HTMLCanvasElement.prototype.getContext` hijack was already replaced with scoped `TU_CanvasOptimizer` in Phase 0.

## Rendering Pipeline (Before vs After)

### Before (Onion Model)
```
applyPostFX call -> underwater(weatherOptimized(weatherTint(experienceOptimized())))
                    ^4 nested function calls, each calling prev.call()
```

### After (With Phase 2)
```
applyPostFX call -> underwater(weatherOptimized(experienceOptimized()))
                    ^3 nested calls (Layer 2 eliminated)
                    + PostFxPipeline infrastructure ready for future linearization
```

## Files Modified
- `part3_game_single (6) (9)(9)(2)(6).html` - Main game file

## Backup
- `part3_phase2_pipeline_isolation.html` - State after Phase 2 completion
