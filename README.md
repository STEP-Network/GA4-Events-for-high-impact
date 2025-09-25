# GA4 High-Impact Ad Event (Minimal Script)

`bufferGA4events.html` now implements a lean, real‑time matcher for GPT + Prebid.js.
There is no buffering, no flush step, and no legacy `SN` alias—events are classified
immediately when a relevant GPT event fires.

## What It Does
1. Listens to GPT `slotRenderEnded` and `slotOnload`.
2. Listens to Prebid `adRenderSucceeded` and `bidWon` (if pbjs present) and keeps a simple per‑element event list.
3. If a GPT render size is 1×1 or missing, it substitutes the latest matching Prebid size (by element id, with a fuzzy includes fallback).
4. Classifies each render instantly into:
   - topscroll (path or element id contains `topscroll` / token `div-gpt-ad-topscroll`)
   - interstitial (path contains `interstitial` OR onload for such slot)
   - midscroll (effective size in allowlist: 300x240, 300x210, 970x570, 970x550, 970x540)
5. Immediately invokes `stepnetwork.trackAdEvent(kind, payload)` for every matched kind.

## Public API
`window.stepnetwork.trackAdEvent(kind, payload)`

You replace the body with your GA4 (or dataLayer) send logic. Example:
```js
window.stepnetwork.trackAdEvent = (kind, payload) => {
  gtag('event', 'hi_' + kind, {
    ad_el_id: payload.elId,
    ad_unit_path: payload.path,
    size: payload.effectiveSizeStr || payload.size_str,
    ts: payload.t
  });
};
```

## Config Constants (edit in file)
- `ALLOWED_MIDSCROLL_SIZES`
- `TOPSCROLL_TOKEN`
- `INTERSTITIAL_TOKEN`

Set `window.ayManagerEnv.debug.flags[0] = 'true'` before the script loads to enable the one remaining debug log (fires only inside `trackAdEvent`).

## Integration Steps
1. Include script after GPT and before ads render.
2. Ensure Prebid is on page (optional but recommended for size substitution).
3. Override `trackAdEvent` with your GA4 / analytics logic.

## Notes
- Self‑guards with `window.__SN_GPT_PREBID_BUFFER_ATTACHED__`.
- Gracefully does nothing if Prebid is absent (midscroll still works when GPT reports real size).
- Fuzzy includes fallback lets slightly mismatched element ids still correlate.

## Extending Quickly
- Add more sizes: push into `ALLOWED_MIDSCROLL_SIZES`.
- Add another classification: duplicate the logic in `classifyAndTrack`.

Minimal by design: real‑time classify → call your tracker.