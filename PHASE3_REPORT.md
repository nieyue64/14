# Phase 3 Report: Resilience & Worker Rebuild

## Completion Status: DONE

## Changes Made

### 1. StorageAdapter - Unified Storage with Degradation Flow (Level 4)
- **Problem**: Save operations used ad-hoc try/catch patterns across multiple patches. LocalStorage quota failures were silently swallowed. IDB backup was bolted on as afterthought. There was no unified awareness of which storage backend was active.
- **Fix**: Introduced `TU.StorageAdapter` with:
  - `save(key, value, opts)` - Tries localStorage first, then IDB, with proper error surfacing
  - `load(key, opts)` - Tries localStorage first, then IDB, returns source info
  - Auto-degradation: `full` -> `idb-only` -> `degraded` -> `lite`
  - Reports quota exceeded errors to GlobalErrorBoundary
  - Tracks storage mode and error statistics
  - Registered as `storageAdapter` service in AppServices

### 2. BlockRegistry - ID Mapping & Palette System (Level 4)
- **Problem**: Block IDs were allocated by blindly scanning for unused IDs:
  ```js
  for (let id = start; 255 > id; id++) if (!BD[id] && !used.has(id)) return id;
  ```
  If patch load order changed, the same block name could get different IDs, corrupting existing saves.
- **Fix**: Introduced `TU.BlockRegistry` with:
  - `register(name, id, data)` - Register known block with stable name-to-ID mapping
  - `allocate(name, data)` - Allocate dynamic IDs for patch-added blocks (starting at 200)
  - `getPalette()` - Generate a name-to-ID mapping for save headers
  - `translate(oldPalette, oldId)` - Translate old save IDs to current IDs using palette
  - Auto-initializes from `BLOCK` constants on `game:init:post`
  - Reserves IDs 0-199 for core blocks, 200-255 for dynamic/patch blocks

### 3. WorkerProtocol - Structured Communication (Level 4)
- **Problem**: Worker communication used raw `postMessage`/`onmessage` with no timeout protection. If `generate()` hung due to a race condition or bad seed, the Promise would dangle forever. The `fn.toString()` approach for building Worker source was fragile.
- **Fix**: Introduced `TU.WorkerProtocol` with:
  - Standard message types: `INIT`, `GENERATE`, `PROGRESS`, `RESULT`, `ERROR`, `SYNC_TILES`, `HEARTBEAT`
  - `request(worker, message, timeoutMs, onProgress)` - Promise wrapper with timeout (default 30s)
  - `heartbeat(worker, intervalMs, onDead)` - Liveness monitor with dead-worker callback
  - Timeout errors routed to GlobalErrorBoundary
  - Each request has a unique ID for multiplexed communication

## Architecture Impact

### Before Phase 3
- Save: `try { localStorage.setItem(...) } catch(e) { console.warn('[Catch]',e); }` (silent failure)
- Block IDs: Blind scan allocation, order-dependent
- Workers: Raw postMessage, no timeout, no heartbeat

### After Phase 3
- Save: `StorageAdapter.save()` with explicit degradation chain and error reporting
- Block IDs: `BlockRegistry` with palette mapping in save headers
- Workers: `WorkerProtocol.request()` with timeout, heartbeat, structured messages

## Files Modified
- `part3_game_single (6) (9)(9)(2)(6).html` - Main game file

## Backup
- `part4_phase3_resilience.html` - State after Phase 3 completion
