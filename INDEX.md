# Index

Everything in this repository, what it is for, and where to start.

---

## Start here

| If you want to… | Go to |
|---|---|
| Get the keyboard working | `README.md`, the Setup section |
| Use only some of the rules | `builder.html` — switch off what you do not want |
| Change what a repurposed key does | `builder.html`, or edit the `to` block by hand |
| Work out why a rule is not firing | `README.md` first: FN lock and the two macOS settings account for most of it |
| Understand why a rule is shaped the way it is | `NOTES.md` |

---

## The files

| File | What it is | Read it when |
|---|---|---|
| `README.md` | Setup and the keymap | First. It is the only file you must read |
| `retro108-macmap.json` | The rule set. Twenty-two manipulators | You are pasting it into Karabiner |
| `builder.html` | An offline page listing all twenty-two rules with a switch on each, and a choice of action for the ones whose output is a matter of taste. Assembles the file as you go | You want some of the rules but not all |
| `NOTES.md` | The lab notebook: the reasoning, the traps, and the deliberate non-fixes | Before changing anything, and any time something surprises you |
| `Retro-108-Mechanical-Keyboard.pdf` | 8BitDo's own booklet, kept here so the `FN` combinations and the key legends are to hand | You need the layout, the `FN` row, or to check what a key sends on Windows |
| `LICENSE` | MIT | — |

---

## What the rule set contains

Twenty-two manipulators. Every one carries `device_unless is_built_in_keyboard`, so a
laptop keyboard is left alone.

| Group | Rules | What it covers |
|---|---|---|
| Screenshots and capture | 5 | The Xbox Game Bar F-row, turned into the macOS equivalents. Needs FN lock engaged |
| The padlock key | 1 | A `simultaneous` rule, because the key sends `l` and left Command together |
| Navigation and editing | 3 | Terminal-compatible line movement, and a forward delete macOS understands |
| Keys macOS ignores | 3 | Scroll Lock, Pause and Insert sent as `F14`–`F16` so they become assignable |
| F-keys silenced | 5 | `F5`–`F8` and `F12` disabled so macOS cannot claim them |
| Num Lock | 1 | Starts the screensaver |
| Modifier swaps | 4 | Windows layout to Mac layout |

**Order matters.** Karabiner takes the first manipulator that matches, so a broad rule above
a narrow one hides it. Keep the array in the order it ships in; the builder preserves it.

---

## Viewing `builder.html`

GitHub shows HTML files as source rather than rendering them. Two ways round it:

- **Download the file and open it in a browser.** It is self-contained and works offline.
- **Enable GitHub Pages** for this repository (`Settings → Pages`, source: deploy from the
  default branch). The builder is then live at
  `https://<owner>.github.io/<repo>/builder.html`.

---

## A sibling project

The same treatment for the 8BitDo Retro R8 Mouse lives in `retror8-macmap`. Worth knowing if
you own both: that rule set is scoped with `device_if` to the mouse alone, while this one
uses `device_unless is_built_in_keyboard`, which excludes a laptop keyboard and nothing else.
If a rule with `f16` through `f19` in its `from` is ever added here, the mouse would start
firing it. Nothing collides today.
