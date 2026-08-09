# Beat Hive — bug log and eval

A record of what broke, why, and what now catches it. Written because several
of these reached the user's phone when a cheap check would have caught them
first, and because the same *kinds* of mistake kept recurring.

## Scoreboard

| # | Bug | Rounds to find | Root cause class | Caught now by |
|---|---|---|---|---|
| 1 | Dial CSS vanished | 1 | Positional edit deleted code | CSS-rule presence check |
| 2 | `discBtn` used before declaration | 1 | Temporal dead zone | Stub-DOM execution |
| 3 | Whole app landed on the piano page | 1 | Throw aborts the whole `<script>` | Execution + script isolation |
| 4 | `recBtn` null | 1 | `getElementById` on a detached node | Stub models DOM connectivity |
| 5 | Pads fell back to red | 1 | Original `switchDjBank` deletes `--pc` | Pad-colour integrity check |
| 6 | Pad grid overflowed right | 1 | `aspect-ratio` vs `grid-template-rows` | — (visual) |
| 7 | Dial label sat below the band | 1 | Padding not folded into band offset | qa-validator agent |
| 8 | `settle()` fired twice | 1 | Own `scrollTop` write re-entered it | qa-validator agent |
| 9 | Dancer 2 never appeared | 1 | `seed()` never called `onPick` | Roster-derived defaults check |
| 10 | **Mic silent** | **4** | **Four different causes, stacked** | On-screen diagnostic |

## The mic, in detail

This one cost the most and deserves the write-up. Four separate faults, each
masking the next:

1. **Node not retained.** `ctx.createMediaStreamSource(s).connect(tap)` created
   inline — no strong reference, garbage-collected, silently stops producing.
2. **Cross-context connect.** The source was built when the mic button was
   tapped, but `djAudioCtx()` recreates the context if it was ever closed.
   Connecting across two contexts throws `InvalidAccessError`.
3. **Insecure origin.** On `http://192.168.x.x`, `navigator.mediaDevices` is
   *undefined*. No audio-graph fix could ever have mattered.
4. **Suspended context.** Granted, live track, meter reading flat zero — the
   AudioContext was still suspended because the mic was enabled before Play.

**The actual lesson is not about Web Audio.** Faults 1 and 2 were fixed blind,
on theory, and neither was the live problem. What ended it was spending two
minutes adding an on-screen readout — secure-context, API presence, track
state, live peak, context state. One screenshot then identified fault 3, and
the next identified fault 4, immediately and unambiguously.

> When you cannot see the failing environment, instrument it. Do not theorise
> at it. Every round spent guessing costs a deploy cycle and the user's
> patience; a diagnostic costs minutes and pays out on the first screenshot.

## Recurring failure patterns

**Deletion is invisible to delta checks.** Bugs 1 and the `#rsBottom` CSS loss
both came from editing by string position (`s[:start] + new + s[end:]`) or a
greedy regex, silently removing code between the anchors. Early QC only
verified that new things were *present* — never that old things still were.
The gate now validates the whole rendered state.

**Parsing is not executing.** Bugs 2, 3 and 4 were all syntactically perfect.
`new Function(src)` proves nothing about TDZ, null dereferences, or ordering.
`tools/exec-check.js` runs the script against a stub DOM, and that stub
deliberately models connectivity: `getElementById` returns `null` for elements
created but never inserted, which is exactly where bug 4 hid.

**A safety net in the same `<script>` is not a safety net.** Bug 3: the
`setMode('dj')` fallback sat outside the IIFE but inside the same block. An
uncaught throw aborts the entire block, so it never ran. It needs its own
`<script>` tag — each block fails independently.

**Assertions rot.** The QC hardcoded `who2='girl'`, so an intentional default
change registered as a failure. Rewritten to derive the expected dial index
from the roster, it now validates the real invariant (dial seed = roster index
+ 1, accounting for the "none" entry) instead of a specific name.

## Bugs inherited from the original app

Two were pre-existing in `pitch-trainer/index.html`, not introduced here:

- **`switchDjBank()` calls `removeProperty('--pc')`**, permanently deleting the
  inline pad colours. Leaving DRUMS once turns all 16 pads the CSS-default red.
  Still present upstream.
- **iOS fires synthetic mouse events after touch**, so a desktop mouse fallback
  double-fired the bank switch on every swipe.

## What the gate runs

`node tools/qc.js` — 15 checks, exit 1 on failure:

- every `<script>` block parses (not just the last)
- every CSS rule the JS depends on exists
- every id the JS creates is styled
- no orphan layout CSS for removed elements
- all sprites 16x16 and within palette range
- no two elements collide on a layout slot (declared exceptions aside)
- pad colours survive a bank switch
- swipe is touch-only
- defaults match the roster
- **the script executes against a stub DOM without throwing**

It has since caught: two stale `order` rules, a regex that ate `#rsBottom`'s
CSS, an unstyled `rsPianoRange`, and its own rotted assertion.
