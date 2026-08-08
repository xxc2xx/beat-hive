# Beat Hive

A single-screen beat maker for kids, built as a PWA. Tap pads, pick a groove,
pick who dances on the LCD, record yourself over the top, and share it.

**Play it:** https://xxc2xx.github.io/beat-hive/

## What's in it

- **16 pads**, swipeable across four banks — Drums, Melody, Chop, Piano
- **18 grooves** — Hip-Hop, Boom Bap, House, Disco, Techno, Trap, Funk, Rock,
  Reggae, Dancehall, DnB, Breaks, Jersey, Garage, Afro, Amapiano, Samba, Bossa —
  plus Shuffle, which rolls a new one every few bars
- **Dot-matrix LCD** with pixel dancers. Each moves to the genre *and* has its
  own personality: Claw'd hops, the T-Rex breakdances, Momo the cat mostly naps,
  Cadet tries hard and fails
- **Record + share** — capture the mix (with your voice, if you enable the mic)
  and send it straight to the iOS share sheet
- **Step sequencer** and a scratch deck, swipeable to save space

## Running locally

```
python3 -m http.server 8000
# then open http://localhost:8000
```

The microphone needs a **secure context**. `localhost` counts; a plain
`http://192.168.x.x` LAN address does not, and iOS additionally refuses mic
access behind a self-signed certificate. Use the GitHub Pages URL for mic
testing on a phone.

## Validation

`tools/qc.js` is the gate. It validates the whole rendered state rather than the
last diff, because most regressions here have been things *removed* or detached
rather than added:

```
node tools/qc.js
```

It checks that every script block parses, that every CSS rule the JS depends on
still exists, that all dancer sprites are 16x16 and within their palette, that
no two elements collide on a layout slot, and — via `tools/exec-check.js` — that
the script actually *executes* against a stub DOM. That last one models DOM
connectivity, so `getElementById` returns `null` for elements created but never
inserted; two shipped bugs hid in exactly that gap.

`tools/qc-hook.sh` wraps it as a Claude Code `PostToolUse` hook.

## Credits

Built with Claude Code. The dancing crab is Clawd, Anthropic's Claude Code mascot.
