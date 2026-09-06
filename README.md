# 8BitDo Retro108 MacMap with Karabiner

A [Karabiner-Elements](https://karabiner-elements.pqrs.org/) rule set that makes the
8BitDo Retro 108 mechanical keyboard behave properly on a Mac — Windows-layout modifiers
swapped, the Xbox Game Bar F-row turned into macOS screenshot and recording shortcuts, the
padlock key wired to Lock Screen, and the dead keys given jobs.

**Every rule is scoped with `device_unless is_built_in_keyboard`, so your MacBook's own
keyboard is untouched.**

There are two files:

| File | What |
|---|---|
| `retro108-macmap.json` | The rule set. A bare `{description, manipulators}` object |
| `NOTES.md` | The lab notebook — why each rule is shaped the way it is, the diagnostic traps, and the things that turned out to be dead ends |

---

## Setup

Three things have to be right. Rules alone are not enough — two macOS settings and one
keyboard hardware state matter just as much, and each one produced a symptom that looked
like a broken rule.

### 1. macOS settings

**`System Settings → Keyboard → "Press 🌐 key to:"` → `Do Nothing`**

The board's `Fn` reaches macOS as the Globe key. The default Globe action ("Show Emoji &
Symbols") fires before Karabiner sees it, so `Fn`+`Home` opens the emoji picker instead of
running its rule. Note this setting may be global rather than per-keyboard depending on
your macOS version — if your laptop's Globe key stops opening the emoji picker, that is
why.

**`System Settings → Keyboard → Keyboard Shortcuts → Function Keys` → leave OFF**

This rule set disables `F5`–`F8` and `F12` inside Karabiner, which means macOS never sees
those keys and cannot impose its own meanings on them. Turning "Use F1, F2, etc. keys as
standard function keys" ON is therefore unnecessary — and on macOS versions where the
panel has no per-device dropdown it is global, which strips the media and brightness
functions from your laptop's own F-row.

Neither of these is the **Modifier Keys** panel. The modifier swaps in this rule set live
entirely in the JSON; nothing needs configuring there.

### 2. FN lock on the keyboard

See the section below — this is a firmware state, not a setting, and four rules depend on
it.

### 3. Import the rules

1. Open `retro108-macmap.json` and copy the whole file.
2. `Karabiner-Elements → Complex Modifications`.
3. **Remove any existing entry for this keyboard first.** Adding does not replace, so
   entries stack and the older one matches first — a stale copy will quietly win over the
   one you just added.
4. **Add your own rule** → paste → Add.

If a rule does not seem to fire, check what Karabiner actually has loaded:

```bash
python3 -c "
import json,os
d=json.load(open(os.path.expanduser('~/.config/karabiner/karabiner.json')))
for p in d['profiles']:
    if not p.get('selected'): continue
    for r in p['complex_modifications']['rules']:
        for i,m in enumerate(r['manipulators']):
            print(i, m.get('description'))
"
```

---

## REQUIRED: FN lock must be engaged

The Retro 108's F-row has two states, toggled by **`Fn` + `Esc`** (padlock printed on the
Esc keycap; firmware-stored, so it survives reboots and follows the board to another
machine). Do not reason about which one the keyboard calls "locked" — the polarity is
easy to get backwards. Judge it by behaviour instead.

**Test: press bare `F1`, holding nothing.**

| Result | State | Verdict |
|---|---|---|
| A screenshot fires (raw shows `print_screen` + `left_command` + `left_option`) | **engaged** | Correct. Leave it here. |
| Nothing, or brightness (raw shows plain `f1`) | plain-F-key mode | Press `Fn`+`Esc` once. |

Four manipulators — the F1-F4 screenshot/recording/F13 rules — fire *only* with FN lock
engaged. In plain-F-key mode they are dead.

The dot printed on `F3` is a **record button** — on Windows this row drives Xbox Game
Bar: F1 screenshot, F2 clip last 30s, F3 start/stop recording, F4 mic mute. What each key
does on the Mac is in the Keymap.

## REQUIRED: two macOS settings

**1.** `System Settings -> Keyboard -> "Press key to:"` -> **Do Nothing**. The board's Fn
arrives as the Globe key, and the default Globe action ("Show Emoji & Symbols") fires
before Karabiner sees it — `Fn`+`Home` opens the emoji picker instead of running its rule.

**2.** `System Settings -> Keyboard -> Keyboard Shortcuts -> Function Keys`,
8BitDo selected in the device dropdown, **"Use F1, F2, etc. keys as standard function
keys" ON.** Without it macOS imposes its own meanings on the plain `f5`-`f8` (keyboard
backlight, rewind, play), which collides with the board's own legends.

Both are distinct panels from **Modifier Keys**, and neither has anything to do with the
modifier swaps — those live entirely in the rule JSON.

---

## Keymap

Everything the rule set changes, generated from the config 2026-09-05. Anything not
listed here is untouched and behaves as a normal Mac key.

### Function row (FN lock engaged)

| Key | Board sends | Result |
|---|---|---|
| `F1` | `print_screen`+cmd+opt | Screenshot area -> file (crosshair) |
| `F2` | `g`+cmd+opt | Capture region -> **clipboard** (crosshair) |
| `F3` (record dot) | `r`+cmd+opt | Screenshot / recording toolbar (`Cmd+Shift+5`) |
| `F4` | `m`+cmd+opt | `F13` — silent, free slot (Discord push-to-talk) |
| `F5` `F6` `F7` `F8` (bare) | plain | **disabled** (`vk_none`) |
| `F9` `F10` `F11` | consumer media codes | rewind / play-pause / forward — no rule needed |
| `F12` (bare) | plain | **disabled** (`vk_none`) |
| `Fn`+any F-key | plain `f1`-`f12` | Literal function keys, carrying their **macOS** meanings (Fn+F5 = complete word, Fn+F6 = cycle focus). The disable rules match bare presses only. Verified 2026-09-05 |

### Navigation cluster and specials

| Key | Result |
|---|---|
| `PrtScrn` (alone) | Screenshot entire screen (`Cmd+Shift+3`) |
| `Scroll Lock` | `F14` = brightness down |
| `Pause` | `F15` = brightness up |
| `Insert` | `F16` — free hotkey slot |
| `Delete` (forward) | Forward delete |
| `Home` / `End` | Native macOS — top / bottom of document or page. No rule |
| `Fn`+`Home` / `Fn`+`End` | Start / end of line, terminal style (`Ctrl+A` / `Ctrl+E`) |
| `Num Lock` | Start screensaver |
| Padlock key | Lock screen (`Ctrl+Cmd+Q`) |
| Mute key, Calculator key, volume knob | Native, no rule |
| `Fn`+`Esc` | FN-lock toggle (firmware). No rule — deliberately left alone |
| `Fn`+arrows | Native macOS: Home / End / Page Up / Page Down |
| `Page Up` / `Page Down`, numpad, Caps Lock, arrows | Native, verified working |

### Modifiers

| Physical key | Acts as |
|---|---|
| Left `Ctrl` | Control (unchanged) |
| Left `Win` | Option |
| Left `Alt` | **Command** |
| Right `Alt` | **Command** |
| Right `Ctrl` | Option |
| `A` / `B` Super Buttons | Firmware only — invisible to macOS, programmable with the star key |

### Not verified

Verified working 2026-09-05: Page Up / Page Down, the full numpad, Caps Lock, arrow keys,
right Shift (confirmed in EventViewer; no typing test will exercise it if you shift with
one hand only).

Still untested:

- A Menu / application key, if this board has one
- `Fn` combinations beyond `Fn`+`Home`, `Fn`+`End`, `Fn`+`Esc` and the printed F-row.
  Bluetooth pairing slots are usually `Fn`+number on this kind of board.

**Bare `Home`/`End` were originally mapped to `Cmd+Left`/`Cmd+Right`** by the forked
gist, to imitate Windows line behaviour. That made them dead in a browser, where there is
no text cursor. Both rules were removed 2026-09-05: line behaviour already lives on
`Fn`+`Home` / `Fn`+`End`, so the bare keys are better left native, where they scroll to
the extremes of the document.

---

## Repurposing a disabled key

`F5`, `F6`, `F7`, `F8` and `F12` are mapped to `vk_none`, which consumes the key and emits
nothing. They are disabled rather than left alone because plain `f5` is macOS's "complete
word" and `f6` cycles keyboard focus — app-level defaults that the standard-function-keys
setting does not suppress.

To give one of them a job, replace its `"to"` block. Three shapes cover almost everything:

```json
"to": [ { "key_code": "f13" } ]
"to": [ { "key_code": "5", "modifiers": ["left_command", "left_shift"] } ]
"to": [ { "shell_command": "open -a Calculator" } ]
```

Then re-import: remove the existing entry in Karabiner and paste the updated rule. Editing
pasting again does nothing unless you remove the old entry first.

Leave the `"from"` block alone. It deliberately carries no `modifiers` key, which means it
matches the **bare** key only, so `Fn`+that key still passes through as a literal function
key. Key-code names come from Karabiner's list, not from the character a key types —
`f13`, `left_arrow`, `delete_or_backspace`, `escape`.

**Never disable a key by mapping it to `F13`-`F20`.** Those are not inert: terminals
translate them into escape sequences, so the key inserts a stray character in a text field
and reads as history-recall in a TUI. Use `vk_none`.

**Other free slots:** `F4` sends `F13`, `Insert` sends `F16`, and the onboard A/B Super
Buttons can carry any chord you record onto them.

---

## Known collision: `Cmd`+`Option`+G / R / M

**If one of these three shortcuts stops working, this is why.**

Rules 2-4 (the F2, F3 and F4 keys) match `left_command` + `left_option` + `g`/`r`/`m`.
After the modifier swaps, pressing what feels like `Cmd`+`Option` on this board produces
exactly that combination, so typing it by hand triggers the F-key rule instead of reaching
the app:

| You press | You get instead of the app's shortcut |
|---|---|
| `Cmd`+`Opt`+`G` | Capture region to clipboard |
| `Cmd`+`Opt`+`R` | Screenshot / recording toolbar |
| `Cmd`+`Opt`+`M` | `F13` (so macOS "Minimize All Windows" does nothing) |

**`Cmd` alone is unaffected.** `Cmd+R` (refresh), `Cmd+G` (find next) and `Cmd+M`
(minimize) all require `left_command` only, do not match these rules, and work normally.
The collision needs *both* modifiers.

**The fix, if you ever need it:** convert those three rules to `simultaneous` matches with
a 20 ms threshold, exactly as the padlock key works — the board emits key-then-modifiers
within one millisecond, while a human presses modifiers first with a gap of ~200 ms, so
timing separates them cleanly. See the padlock entry in [NOTES.md](NOTES.md) for the shape.

**Why it has not been applied:** `simultaneous` buffers the trigger key for up to 20 ms,
and these triggers are the letters `g`, `r` and `m`. That cost falls on every keystroke
and risks garbling fast rollover ("gr", "rm"), traded against three shortcuts that are
rarely used — one macOS-level, two app-specific. Judged not worth it for a fast touch
typist on 2026-09-05. Apply the fix only if the collision is hit in practice.

Related, unresolved: `delete_forward -> fn+delete_or_backspace` (inherited from the fork)
is probably redundant, since macOS handles the `delete_forward` key code natively. Not
verified, and left in place because removing a working key to prove a point is a bad trade.
---

## Credits and licence

Based on [breadbored](https://gist.github.com/breadbored)'s gist, [*Tutorial: Remapping the
8BitDo 108-key Keyboard for MacOS and Karabiner*](https://gist.github.com/breadbored/6dd26c7e2a201c8fae2282d82c5d1ac4),
which got the hard part right — working out that this board emits Xbox Game Bar chords
rather than plain function keys, which is the insight the whole F-row depends on.

This version has since been substantially rewritten. Changes from the original: the `Cmd+Ctrl+Shift+5` screen-recording mapping was replaced
(that is not a real macOS shortcut and the key silently did nothing); `Fn+End` was missing
its `fn` modifier; the bare `Home`/`End` rules were removed so those keys behave natively;
the lock key was rebuilt as a `simultaneous` rule because no modifier-based rule can work
for it; `F5`–`F8` and `F12` were disabled; the screensaver was moved off `Fn`+`Esc`; and
every manipulator was scoped away from the built-in keyboard.

MIT licensed — see `LICENSE`.
