# Hover away from the visible cursor

Use this after the live-testing ownership/capability gates when toolbar or popup hover contradicts the pointer's visible position.
First distinguish a Firefox-only hover reaction from two visible Windows cursors, delayed cursor drawing, or system-wide input mismatch.
A Firefox pointer trace alone neither diagnoses the latter cases nor establishes another physical mouse, remote control, or malware.

## Capture before changing CSS

1. Read the matched reveal rule and native popup implementation for the exact build. Separate `:hover`, `:focus-within` and popup state.
   Do not test a declaration already effective natively, or assume a popup has an open-state CSS selector/particular DOM parent.
2. During one user/native reproduction, record `pointerover`, `pointerout` and `pointermove`: timestamp, `pointerId`, `pointerType`,
   `isPrimary`, `isSynthesized`, screen/client coordinates, target and `explicitOriginalTarget`, ancestor hover, popup state and toolbar style.
   Preserve transitions while coalescing repetitive moves, and bound the trace lifetime. An empty trace does not explain the symptom.
3. Correlate physical movement with pointer IDs. Multiple IDs alone are normal for some input devices; neither ID 0 nor ID 1 has a fixed role.
   A candidate stale pointer stays at unrelated coordinates and reactivates hover while the observed physical pointer moves elsewhere.
   `isSynthesized: false` does not prove a fresh physical event or identify who originally registered the pointer.

Firefox 157's `PresShell::ProcessSynthMouseMoveEvent` revisits retained hover-capable pointer states after layout changes [1].
This can expose a stale tab-strip position when autoscroll changes layout. Prove the divergent event in the trace before blaming CSS.

## Targeted session recovery

Use only with explicit approval for a one-time state correction, a trace identifying the stale ID, and exact-build source/API checks.
A CSS-only request does not authorize a JavaScript state correction. Do not guess an ID, clear all pointers, or install a persistent workaround.

Firefox 157 provides chrome-only `Window.synthesizeMouseEvent`; `mousecancel` represents a disappeared device [2].
`nsContentUtils::SynthesizeMouseEvent` maps it to `eMouseExitFromWidget` with `ePlatformTopLevel` in the parent process [3].
`PointerEventHandler::UpdatePointerActiveState` removes that ID, but a test-flagged exit does not remove a non-test pointer [4].
Thus the approved correction below deliberately uses `isDOMEventSynthesized: false`; it is not a routine test-input default.

Given the owned chrome window `w` and a freshly validated `stale` record (`id`, `screenX`, `screenY`), dispatch once:

```javascript
w.synthesizeMouseEvent(
  "mousecancel", stale.screenX - w.mozInnerScreenX, stale.screenY - w.mozInnerScreenY,
  { identifier: stale.id, buttons: 0 }, { toWindow: true, isDOMEventSynthesized: false }
);
```

Its boolean return reports event cancellation (`preventDefault`), not recovery success. Verify the resulting pointer exit and behavior.
Retest autoscroll over the page, normal tab-strip hover, and return to the page using physical input; obtain visual confirmation separately.
Check that the stale ID no longer produces events during those cases, preserve tabs/selection/layout, and remove tracing before listener cleanup.
Report this as session-only recovery. Its success does not establish the pointer's original creator, permanent prevention, or a Windows-wide fix.

## Exact source basis

These links are Firefox 157 references, not promises about another version or derived build; inspect the target before use.

[1]: https://github.com/mozilla-firefox/firefox/blob/FIREFOX_157_0_RELEASE/layout/base/PresShell.cpp#L6036-L6056
[2]: https://github.com/mozilla-firefox/firefox/blob/FIREFOX_157_0_RELEASE/dom/webidl/Window.webidl#L601-L647
[3]: https://github.com/mozilla-firefox/firefox/blob/FIREFOX_157_0_RELEASE/dom/base/nsContentUtils.cpp#L10396-L10553
[4]: https://github.com/mozilla-firefox/firefox/blob/FIREFOX_157_0_RELEASE/dom/events/PointerEventHandler.cpp#L465-L478
