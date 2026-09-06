# 8BitDo Retro108 MacMap — notes

The reasoning behind the rule set in this repo: why each rule is shaped the way it is,
which diagnostic methods actually work, and the dead ends. Setup instructions are in
[README.md](README.md).

Most of what follows was learned the expensive way over two sessions in September 2026.
The traps are listed because every one of them produced a symptom that looked like a
broken rule and was not.

---

## Hard-won facts

**Karabiner runs the copy you pasted in, not the file you pasted it from.** To change the
rules, update that entry — use the edit button on it, or delete it and add a fresh one.
Nothing you do to the JSON file afterwards reaches Karabiner on its own.

The symptom when this is missed is confusing enough to waste hours: some rules work and
others do not, because what is loaded is an older version of the same set — different
ordering, missing modifiers, rules absent entirely. It reads as several unrelated bugs
rather than one out-of-date copy.

**Verify what is actually loaded** before theorising about a rule that will not fire:

```bash
python3 -c "
import json,os
d=json.load(open(os.path.expanduser('~/.config/karabiner/karabiner.json')))
for p in d['profiles']:
    if not p.get('selected'): continue
    for r in p['complex_modifications']['rules']:
        for i,m in enumerate(r['manipulators']):
            f=m.get('from',{})
            print(i,m.get('description','')[:44],'| from:',f.get('key_code'),'| mand:',f.get('modifiers',{}).get('mandatory'))
"
```

**Order within a rule set matters, and specific must precede general.** A plain
`key_code: end` manipulator placed above `end` + `mandatory: ["fn"]` makes the second
unreachable — the general one matches first and wins. The same inversion on `home`
masked itself, because plain `Home → Cmd+Left` and `Fn+Home → Ctrl+A` both land the
cursor at the start of the line, so the rule looked like it was working when it was not.

**The padlock key needs a `simultaneous` rule, not a modifier rule.** This one key took
most of two sessions and roughly ten attempts. Why no `mandatory` modifier ever worked:

| Press | Raw event order | Gap |
|---|---|---|
| Padlock key | `l` **first**, then `left_command` | same millisecond |
| Human typing `Cmd+L` | `left_option` first, then `l` | ~200 ms |

Two independent problems fall out of that:

1. The padlock emits its key code **before** its modifier, so when `l` is evaluated no
   modifier flag is set yet and `mandatory: ["left_command"]` cannot match.
2. On this Windows-layout board the key in the **Cmd position physically sends Alt**. A
   typed `Cmd+L` arrives as `left_option`, the swap rule converts it to `left_command` on
   the modifier's own key-down, and by the time `l` arrives the state genuinely *is* Cmd —
   so a `left_command`+`l` rule fires on every real `Cmd+L`.

Both presses look identical to any modifier-based rule, reached from opposite directions.
What separates them is **timing and order**, so the rule matches on those instead:

```json
"from": {
  "simultaneous": [ { "key_code": "l" }, { "key_code": "left_command" } ],
  "simultaneous_options": { "key_down_order": "strict", "key_up_order": "insensitive" },
  "modifiers": { "optional": ["any"] }
},
"parameters": { "basic.simultaneous_threshold_milliseconds": 20 }
```

`strict` order requires `l` before the modifier, which a human never does; 20 ms excludes
any human gap. Cost: a plain `l` keystroke is buffered up to 20 ms while Karabiner waits
for a possible modifier. Drop the threshold to 10 if that is ever perceptible.


**Complex modifications are NOT device-scoped by default.** Unlike Simple
Modifications (which have a target-device dropdown), a complex rule applies to *every*
attached keyboard unless the manipulator carries a `device_if`/`device_unless`
condition. This is why the swaps need the `is_built_in_keyboard` guard.

**Documented modification order** (from Karabiner's event-modification-chaining page):
`1. Catch events from hardware → 2. Apply Simple Modifications → 3. Apply Complex
Modifications`.

**Manipulators do not re-process their own output.** A paired bidirectional swap
(`left_command→left_option` *and* `left_option→left_command` in one list) is safe and
does not cancel itself. Proof: pqrs-org's own maintained `swap_cmd_option_unless_builtin.json`
ships exactly that pattern.

**A complex-modifications file needs the `{"title": ..., "rules": [...]}` wrapper.**
A bare `{description, manipulators}` object is valid JSON but an invalid Karabiner
file. Pasting one into a JavaScript context yields
`SyntaxError: unterminated statement (line 2)` — that error means the file reached a
JS engine, never Karabiner, which reports bad configs as `Invalid data in <file>.json`.

---

## Diagnostic method

**Karabiner-EventViewer is ground truth** for "what does this key actually send" —
the macOS equivalent of `xev` on the Linux box. Two traps, both of which cost hours:

- **EventViewer does not see your rules, and your rules do not protect EventViewer.**
  In raw capture the events shown are pre-modification, and the frontmost EventViewer
  window receives the *unmodified* chord. On 2026-09-05 this looked like a broken rule
  for over an hour: `F4` sends `m`+cmd+opt, and pressing it inside EventViewer minimised
  that window, because macOS got the raw `Cmd+Opt+M`. In any normal app the rule was
  firing correctly the whole time. **Never judge whether a rule fired from inside
  EventViewer** — use a real app, or the isolation test below.

- Its **name** column shows *key codes*; other columns show the *character*. `!` is a
  character (Shift+1), never a key code. `l` and `!` look nearly identical in the UI
  font — read carefully, and copy the raw row rather than describing it.
- **EventViewer has two capture modes, and only one is ground truth.** *Capture Input
  Events* shows events **after** macOS and Karabiner processing. *Capture Raw Input
  Events*, with the 8BitDo selected in its device filter, shows what the board actually
  emits before anything touches it. A rule's `from` matches the raw layer, so raw is
  the view to design against. Judging from the processed view is what made the lock key
  take many attempts, and what produced the premature "Fn is dead" call.

- **Worked example (2026-09-05):** F1 reads as `f1` in raw on the 8BitDo, but as
  `display_brightness_decrement` in Capture Input Events. The board sends a true F1;
  macOS converts it downstream via System Settings → Keyboard → Keyboard Shortcuts →
  **Function Keys** ("Use F1, F2, etc. as standard function keys"), which is
  per-keyboard on recent macOS. This is a different panel from Modifier Keys and is
  not related to the modifier swaps.

**The isolation test** — splits a broken rule in one keypress. Swap the `to` for:

```json
"to": [ { "shell_command": "open -a Calculator" } ]
```

Calculator opens → `from` matches, problem is in `to`. Nothing → `from` isn't
matching, and `to` is irrelevant.

---

## Super Buttons — onboard, no software

Firmware-level, so fully functional on macOS.

**To assign:** press **★** (Fast key mapping) → **type the combo** (up to 6 keys
simultaneously) → **press the Super Button**. Combo first, button second.

- Cancel one mapping: press ★, then that Super Button.
- Clear all: hold ★ for 5 seconds.
- Mapping mode auto-exits after 60 seconds idle.
- Key mapping indicator blinks rapidly when a mapped button is pressed.
- Stored in the keyboard; survives reboots and follows it to any machine.
- Factory reset (Pair + ★ + Profile, 5 sec) wipes mappings, pairings and swaps.

**The Super Buttons are ON the board** — the two keys labelled **A** and **B**, sitting
just right of the spacebar. They are firmware controls: they send nothing over USB, so
Karabiner and EventViewer cannot see them at all. That is expected, not a fault.

The two **3.5mm jacks** on the back take *additional*, external Super Buttons / 8BitDo
Keyboard Extensions and program identically. **They are not audio jacks.** The external pedals and the onboard A and B are all available.

**To reach A/B from Karabiner**, use the chord trick: record an unused combination
(`Ctrl+Alt+Shift+F9`) onto A with Fast Key Mapping, then catch that chord in a rule. The
button then triggers anything a manipulator can express, including a `shell_command`.

**Fast Key Swap** (`Win`+`Alt`+`★` swaps those two in firmware; works on
Ctrl/Win/Alt/Shift) — **do not use.** The swap is handled in Karabiner; doing both
cancels out.

**Useful pattern:** Fast Key Mapping can only record keys that physically exist, so
F13–F16 can't be recorded directly. Instead record an absurd chord
(`Ctrl+Alt+Shift+F9`) onto a Super Button, then catch that chord in Karabiner and map
it to anything, including a `shell_command`.

## Ultimate Software V2 — not available for this board on macOS

Retro 108 support landed in the **Windows v1.10** build. The macOS build's keyboard
list covers only the Retro 87 series and Retro 68-N40. So the 3 custom profiles, the
macro editor, and firmware updates all require a Windows machine (a VM with USB
passthrough works).

**Not critical.** 8BitDo's keyboard-side firmware changelog is quality-of-life work
("optimized macro stability", language/usage fixes) on features unused here. Revisit
only if Bluetooth or 2.4G starts misbehaving.

The **Profile button** is effectively inert without that software — the manual
documents only its existence and the factory-reset combo, nothing more. Profiles and
onboard key mappings are stored separately.

---

**Scroll Lock and Pause land on brightness, by design not accident.** The rules map them
to `f14` / `f15`, and macOS treats **F14 / F15 as the legacy brightness-down / -up keys**.
The board genuinely sends `scroll_lock` and `pause`; Karabiner converts; macOS interprets.
Left as-is deliberately — it is free brightness control on a board that has none.

## Physical layout notes

Top-right cluster past F12: `FN`, `Mute`, `Calculator`, `Screen lock`. Three
unlabeled keys sit between F12 and FN (PrtScrn / ScrollLock / Pause).

Bottom row, right of the spacebar: the two **A** / **B** Super Buttons, then
`right_control` as the farthest-right key. A and B register nothing in EventViewer
because they are firmware controls, and they work correctly on the Mac as shipped —
untouched so far, not ruled out. `right_control` is currently consumed by the
`right_control -> right_option` swap.

Works natively on macOS with no rules: tri-mode connectivity (USB-C / 2.4G dongle /
Bluetooth), volume knob, Mute key, `Fn+F9`/`F10`/`F11` media keys, N-key rollover,
hot-swap sockets.

Mode switch positions: `OFF` = wired, `2.4` = dongle, `BT` = Bluetooth.
Battery 2000mAh, ~200h. Auto-shutdown after 15 min idle (not while wired).

## Still open (as configured here)

Nothing blocking. Only:

- Whether this board has a Menu / application key, and what it sends.
- `Fn`+number and `Fn`+letter combinations were never swept. Likely empty — the board has
  a dedicated Pair button, so Bluetooth slots are probably not on the number row. Worth a
  look only if you need more free chords to map.
- The onboard **A** / **B** Super Buttons and the external 3.5 mm ones are untouched.
  They work as shipped and are available.

Two collisions are known and deliberately **not** fixed — see the collision section.

---

## Sources

- Manual (image-only PDF, 60pp, English on pages 1–7): <https://download.8bitdo.com/Manual/PC-Peripherals/Retro-108-Mechanical-Keyboard.pdf>
- Ultimate Software V2 release notes / device matrix: <https://app.8bitdo.com/Ultimate-Software-V2/>
- Official Karabiner rule repo: <https://github.com/pqrs-org/KE-complex_modifications>