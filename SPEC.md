# Graduated Extinction Timer — Claude Code Spec

## Summary

Build a **single-file HTML/JS/CSS webpage** (`index.html`) that runs a graduated extinction ("controlled comforting") timer using the Hiscock interval schedule. Designed for use by an exhausted parent standing in a dark nursery at 3 am on a phone screen.

No build step. No frameworks. No CDN dependencies. Must work offline after first load. Target: phone portrait (375 × 812 minimum), but should scale up to tablet and desktop gracefully.

---

## Context (why this exists)

This tool supports the graduated extinction protocol for a 6–8 month old who wakes 5+ times/night. The parent enters the room at timed intervals during their child's crying, does a brief check (15–60 s pat + reassurance + leave), then waits the next interval. A physical timer or phone stopwatch works poorly because:

1. Manually setting 2 min → 4 min → 6 min → 8 min → 10 min → 10 min → 10 min is error-prone at 3 am.
2. A bright phone screen disrupts the parent's and child's night vision.
3. Most clock apps are noisy, cheerful, and arousing — the opposite of what's wanted.

The webpage replaces that with a single dark, near-silent, dead-simple interface.

---

## The interval schedule (hardcoded)

**Hiscock controlled comforting progression:**

```
Wake 1, check 1:  2 min
Wake 1, check 2:  4 min
Wake 1, check 3:  6 min
Wake 1, check 4:  8 min
Wake 1, check 5:  10 min
Wake 1, check 6+: 10 min (stays at 10)
```

At each new night waking, the schedule **restarts from 2 min**. This is critical to the protocol — do not carry intervals across wakes.

Represent internally as an array with a "repeat last" flag:

```js
const INTERVALS_MINUTES = [2, 4, 6, 8, 10];
const REPEAT_LAST = true; // checks 6, 7, 8... all use final value
```

---

## UI states

The app has exactly four states. Transitions between them are the only interactions the user ever performs.

### State 1: IDLE (first load / after reset)

- Large centered text: "Ready"
- Sub-text: "Tap to start a check interval"
- Single large button: **START**
- Small footer: current wake number (default "Wake 1"), sound toggle icon, info icon

### State 2: COUNTING DOWN

- Large centered text: MM:SS countdown (e.g. `01:47`)
- Sub-text above: "Interval 1 — 2 min" (or whichever)
- Sub-text below: "Leave the room. Wait for the full interval."
- Single large button: **CANCEL** (returns to IDLE, does not advance interval)

### State 3: INTERVAL ELAPSED (prompt to check)

- Triggered when countdown reaches 00:00
- Large centered text: "Time to check"
- Sub-text: "Brief pat, reassure, leave. Keep it boring."
- Two buttons stacked:
  - **CHECK DONE → NEXT INTERVAL** (advances to next interval, returns to State 2)
  - **BABY ASLEEP — END WAKE** (returns to State 4)

### State 4: WAKE COMPLETE

- Large centered text: "Wake ended"
- Sub-text: "Total: X min over Y checks"
- Single button: **READY FOR NEXT WAKE** (increments wake counter, resets intervals, returns to State 1)

---

## Functional requirements

### Core timer

- Countdown in 1-second ticks using `setInterval` or `requestAnimationFrame` against `performance.now()` for accuracy.
- Do NOT drift: compute remaining time from a stored `endTimestamp = Date.now() + intervalMs`, not by decrementing a counter.
- When tab backgrounded, the timer must remain accurate on return — re-read `endTimestamp` and recompute.

### Elapsed notification

When the countdown reaches 00:00:

- Play a single soft chime (Web Audio API, generated sine wave, ~440 Hz, 200 ms, fade out). User must be able to mute.
- Trigger `navigator.vibrate([200, 100, 200])` if available.
- Flash a subtle background pulse (one cycle, slow — 1.5 s fade up to ~20% brightness and back).
- Advance to State 3.

### Screen Wake Lock

Request `navigator.wakeLock.request('screen')` when entering State 2, release on entering State 1 or 4. Handle the Promise rejection silently — API is not available in all browsers and that's fine.

### Sound toggle

Persist to `localStorage` as `extinction-timer-sound-enabled` (boolean). Default: true. Shown as a small icon in the footer; tapping toggles.

### Wake counter

Increment on transition from State 4 → State 1. Display in footer as "Wake 3". Persist to `localStorage` as `extinction-timer-wake-count` with a timestamp — if last update was more than 12 hours ago, reset to 1 on app open (new night).

### Keyboard shortcuts (for desktop testing; not required on mobile)

- `Space` or `Enter`: advance primary action (START / CHECK DONE / READY FOR NEXT WAKE)
- `Esc`: CANCEL from State 2

---

## Visual design

### Palette (strict)

| Role | Color | Notes |
|---|---|---|
| Background | `#000000` | Pure black. OLED power saving. |
| Primary text | `#7a1a00` | Deep amber-red. Night-vision preserving. |
| Secondary text | `#3d0d00` | Dimmer amber. Used for sub-text and footer. |
| Button fill (idle) | `transparent` | Buttons are outlined only. |
| Button border | `#5a1400` | Subtle. |
| Button text | `#8f2000` | Slightly brighter than primary for affordance. |
| Button active/pressed | `#1a0500` fill | Very subtle highlight. |
| Pulse flash | `#2a0800` | The elapsed-notification background flash. |

**Do not use:**
- Pure white anywhere
- Blue or cyan (highest melatonin suppression)
- Green, yellow, orange at normal brightness
- Any color above ~30% luminance

### Typography

- System font stack: `-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif`
- Countdown: 96px, weight 200 (thin), letter-spacing normal, tabular-nums
- State label: 20px, weight 400, uppercase, letter-spacing 0.15em
- Sub-text: 16px, weight 300, line-height 1.5
- Buttons: 18px, weight 400, uppercase, letter-spacing 0.1em
- Footer: 12px, weight 300

### Layout

- Viewport: full-height flex column, centered content.
- Padding: 24px from screen edges minimum.
- Button height: 56px minimum (WCAG tap target).
- Button width: full width on phone (with 24px margin), max 400px on larger screens.
- Vertical rhythm: 32px gaps between major blocks.

### Interaction feedback

- All transitions: 300 ms ease-out
- No bounces, no springs, no flourishes
- Button press: 100 ms fade to active state, 200 ms fade back
- State transitions: cross-fade content, no slide animations
- **Absolutely no:** confetti, emojis, celebratory animations, sounds beyond the single elapsed chime

### Brightness

- The browser cannot programmatically dim the screen.
- Include a one-line instruction on first load (dismissible, persisted): "Set your phone brightness to minimum before using."

---

## Technical constraints

- **Single file**: all HTML, CSS, and JavaScript in `index.html`. No external resources.
- **No frameworks**: vanilla JS only. No React, no Vue, no build step.
- **No network dependencies at runtime**: must work with WiFi off once loaded.
- **No analytics, no telemetry, no tracking of any kind.**
- **Browser support**: evergreen Chrome, Safari, Firefox. Must work on iOS Safari 15+ and Chrome for Android 100+.
- **File size**: target under 20 KB. Hard cap 50 KB.
- **Accessibility**: semantic HTML (`<button>`, not `<div onclick>`); ARIA live region for state announcements; respects `prefers-reduced-motion`.

---

## Code organisation (within the single file)

```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta viewport>
  <meta theme-color> (set to #000000)
  <title>
  <style> — all CSS, organised: reset → layout → typography → states → animations
</head>
<body>
  <main id="app">
    <!-- Single container whose innerHTML is swapped per state, OR
         four hidden sections toggled by a data-state attribute on <main>.
         Prefer the latter for simplicity. -->
  </main>
  <script>
    // Constants
    // State machine
    // Timer logic
    // Audio (Web Audio API)
    // Wake Lock
    // localStorage persistence
    // Event wiring
    // Init
  </script>
</body>
</html>
```

Use a single state variable (`currentState`) and a `render()` function that reads state and updates the DOM. Do not use a framework's reactivity — imperative DOM updates are fine and keep the file small.

---

## Acceptance criteria

A reasonable test script a human should be able to complete without confusion:

1. Load page. Screen is near-black with dim amber text. "Ready" and a START button visible.
2. Tap START. Countdown shows `02:00` and begins counting down.
3. Wait 2 minutes (or use browser devtools to fast-forward). A soft chime plays, screen pulses briefly, state changes to "Time to check".
4. Tap CHECK DONE. Countdown shows `04:00` and begins counting.
5. Tap CANCEL mid-countdown. Returns to READY state with the timer reset.
6. Run through a full wake: 2 → 4 → 6 → 8 → 10 → 10 (note the repeat). Confirm 6th interval is still 10 min.
7. Tap BABY ASLEEP. See "Wake ended" with total.
8. Tap READY FOR NEXT WAKE. Footer shows "Wake 2". Schedule restarts at 2 min.
9. Toggle sound off. Run a full interval. Confirm no chime, but vibration and visual pulse still occur.
10. Switch to another tab during countdown, wait 30 s, return. Remaining time is correct (not paused).
11. Lock phone during countdown. Unlock 1 min later. Countdown continues correctly.
12. Reload page. State resets to IDLE. Sound toggle preference persists. Wake counter persists if within 12 h of last use.
13. Check `prefers-reduced-motion: reduce` — confirm pulse animation is replaced with an instant color change.

---

## Out of scope (do NOT build)

- User accounts, cloud sync, multi-device support
- Sleep logs, analytics, historical waking data export
- Advice content, tips, or reassurance text beyond the minimal sub-text specified
- Multiple protocols (Ferber, fixed-interval, compressed) — only Hiscock
- Settings screen beyond the single sound toggle
- Push notifications
- PWA manifest / install prompt (nice-to-have, explicitly optional — if adding, keep manifest inline via data URL and ensure it doesn't bloat file above 50 KB)
- Dark/light mode toggle — the app is permanently dark by design

---

## File delivery

- `index.html` — the app
- `README.md` — brief: what it is, how to run (`open index.html` or serve with `python3 -m http.server`), and a one-paragraph note that this is a tool to support a behavioural protocol recommended by a health professional, not medical advice.

---

## Open questions for the implementer

None expected. If anything is ambiguous, default to the simpler option. The overriding principle: a shattered parent at 3 am must be able to use this without reading instructions.
