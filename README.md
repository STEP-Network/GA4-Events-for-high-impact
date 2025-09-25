# GA4 Events High-Impact Support Scripts

This repo currently contains helper script(s) for gathering Google Publisher Tag (GPT) + Prebid.js events, enriching them, and flagging high‑impact ad formats (topscroll, interstitial, midscroll) before sending or inspecting them for analytics (e.g. GA4) or internal debugging.

## File
- `bufferGA4events.html` – Self‑executing script block that:
  1. Buffers raw GPT events.
  2. Buffers selected Prebid events (`adRenderSucceeded`, `bidWon`).
  3. Derives an "effective" rendered size (falls back from 1×1 to matching Prebid size within a short time window).
  4. Classifies renders into three groups: topscroll, interstitial, midscroll (using allowed size list).
  5. Exposes a small API via `window.stepnetwork` (with a legacy alias `window.SN`).

## Core Logic Overview
1. Attach once: Guards with `window.__SN_GPT_PREBID_BUFFER_ATTACHED__`.
2. Collect GPT events: `slotRequested`, `slotResponseReceived`, `slotRenderEnded`, `impressionViewable`, `slotOnload`.
3. Collect Prebid events after pbjs is ready (polling up to ~5s): `adRenderSucceeded`, `bidWon`.
4. For each GPT `slotRenderEnded` (not empty), compute an `effectiveSize`:
   - Use GPT size unless it's missing or 1×1.
   - Otherwise search recent Prebid events (same element id) within 5s to find a better size.
5. Classification rules:
   - Topscroll: any rendered slot whose ad unit path or element id contains `topscroll` or the token `div-gpt-ad-topscroll`.
   - Interstitial: any rendered (or onload) slot whose path contains `interstitial`.
   - Midscroll: any rendered slot whose effective size matches one of an allowlist (e.g. 300×240, 970×570, etc.).
6. `flush()` evaluates current buffered GPT events, returns a summary, and (stub) invokes placeholder handlers (`topscrollEvent`, `interstitialEvent`, `midscrollEvent`).
7. On `pagehide` (modern unload), it automatically flushes and clears buffers.

## Public Surface (`window.stepnetwork`)
- `stepnetwork.gptEvents` (Array) – Raw collected GPT event objects.
- `stepnetwork.pbEvents` (Array) – Raw collected Prebid event objects.
- `stepnetwork.flush(reason?, { clear? })` – Runs classification and returns `{ topscroll, interstitial, midscroll, enriched }`.
  - `reason` (string) optional label.
  - `clear` (boolean) if true, empties buffers after flush.
- `stepnetwork.dump()` – Snapshot `{ gpt: [...], pb: [...] }` without clearing.

Backward compatibility: `window.SN` is still defined (pointing to the same object) if older code references `SN.*`.

## Minimal Usage
1. Make sure this script runs after GPT tag snippet and before/around when ad slots are defined.
2. Ensure Prebid.js is present if you want size fallback enrichment (otherwise it still works with GPT only).
3. Call `stepnetwork.flush('manual')` in the console (or your analytics hook) any time to get current classification.
4. Rely on the automatic flush on `pagehide` for end‑of‑page summary.

Example (in dev console):
```js
// Check what has been captured so far
stepnetwork.dump();

// Run classification manually
const summary = stepnetwork.flush('manual');
console.log(summary.midscroll.map(x => x.effectiveSizeStr));
```

## Custom Event Handling
Replace stub functions inside the script (`topscrollEvent`, `interstitialEvent`, `midscrollEvent`) with real trackers (e.g. push to dataLayer or send GA4 events) if needed. Edit them directly inside `bufferGA4events.html`.

Example modification idea:
```js
function topscrollEvent(payload) {
  dataLayer.push({
    event: 'hi_topscroll_render',
    count: payload.matches.length
  });
}
```

## Configuration Points
- `DEBUG`: Controlled by `window.ayManagerEnv.debug.flags[0] === 'true'`. Change logic or hard‑set to `true` during development for verbose console logging.
- `ALLOWED_MIDSCROLL_SIZES`: Adjust the midscroll detection allowlist.
- Mapping Prebid ad unit code to GPT element id: Edit `mapAdUnitCodeToElementId()` if they differ in your setup.
- Time window for size enrichment: `WINDOW_MS = 5000` inside `effectiveSizeForRender` (adjust if needed).

## Clearing Buffers Manually
```js
stepnetwork.flush('manual-clear', { clear: true });
```
This returns the summary of what existed, then empties both arrays.

## Safety / Idempotency
- The script self‑guards to avoid double attachment.
- Fails gracefully if Prebid is absent (only GPT data collected).

## Extending
- Add new classification sets by copying the pattern in `evaluate()`.
- Add more GPT events by extending the listener list.
- Add more Prebid events via additional `safeOn()` calls if you need deeper diagnostics.

## License / Notes
No license file is included yet. Add one if distributing externally.

---
Simple, focused: buffer, enrich, classify, flush. Modify stubs to integrate with your analytics pipeline.