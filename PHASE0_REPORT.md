# Phase 0 Report: Safety Net & Performance Baselines

## Completion Status: DONE

## Changes Made

### 1. Enhanced TU.Safe with `runAsync` (line ~4999)
- **Problem**: `TU.Safe.run` only caught synchronous errors. Async operations (Worker generation, IDB writes, audio context init) that returned Promises would silently hang or reject without any visibility.
- **Fix**: Added `TU.Safe.runAsync(tag, fn, opts)` that properly awaits Promises and catches rejections. Also routes errors to `TU_Defensive.ErrorReporter` for proper visibility.
- **Impact**: Worker hangs, IDB quota failures, and async save errors will now surface in the error dialog and console.

### 2. Fixed Storage Error Swallowing (ServiceLocator storage adapter, line ~5125)
- **Problem**: `localStorage.setItem()` failures were caught with `catch(e){console.warn('[Catch]',e);}` - critically, `QuotaExceededError` was silently swallowed, causing "zombie bad saves" where users thought their game was saving but it wasn't.
- **Fix**: Storage `.set()` now explicitly detects `QuotaExceededError`, shows a user-facing toast, logs the data size, and **re-throws** so callers (SaveSystem) can handle degradation properly.
- **Impact**: Save failures are now visible to both code and users.

### 3. Fixed `game:init:post` Double Trigger (line ~28892)
- **Problem**: The event `game:init:post` was emitted twice:
  1. At the end of `Game.init()` (line 23913) - the original, correct emission
  2. In a monkey-patch wrapper that re-wrapped `Game.prototype.init` (line 28892)
- **Consequence**: All `game:init:post` listeners (tile logic, machine indexing, weather init, etc.) ran twice, causing first-frame stutter spikes and redundant state initialization.
- **Fix**: Removed the duplicate emission in the monkey-patch. The `__tuGameReadyEvent` flag is preserved to prevent other patches from creating new wrappers.

### 4. Scoped Canvas `getContext` Hijack (line ~5402)
- **Problem**: The canvas state optimizer globally patched `HTMLCanvasElement.prototype.getContext`, intercepting ALL canvas contexts (minimap, offscreen chunks, weather FX, texture generation). This broke V8's hidden class optimization for ALL canvas operations.
- **Fix**: Replaced with `TU_CanvasOptimizer.optimize(ctx)` - a scoped utility that is explicitly called only on the main game Renderer's context. Other canvases retain native performance.
- **Impact**: Minimap, chunk cache, and weather FX canvases no longer go through the interception layer.

### 5. Added GlobalErrorBoundary (new)
- **Purpose**: Centralized fatal/warning error routing that replaces scattered `try/catch(e){}` with structured error reporting.
- **API**: `TU.GlobalErrorBoundary.reportFatal(source, error, meta)` and `.reportWarning(source, msg, meta)`
- **Features**: Rate-limiting, fatal threshold detection, automatic `fatal:error` event emission.

### 6. Added PerformanceBaseline Tracker (new)
- **Purpose**: Collects rolling p50/p95/p99 frame time and tick time measurements for regression detection.
- **API**: `TU.PerformanceBaseline.recordFrame(ms)`, `.recordTick(ms)`, `.getBaseline()`, `.report()`
- **Integration**: Automatically hooks into `game:render:pre/post` and `game:update` events.
- **Auto-report**: In debug mode, prints baselines every 30 seconds.

## Files Modified
- `part3_game_single (6) (9)(9)(2)(6).html` - Main game file

## Backup
- `part0_original_backup.html` - Original file before any changes
- `part1_phase0_safety_net.html` - State after Phase 0 completion
