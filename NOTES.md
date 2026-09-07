# 8BitDo Retro 108 MacMap — notes

The reasoning behind the rule set in this repo: why each rule is shaped the way it is,
which diagnostic methods actually work, and the dead ends. Setup instructions are in
[README.md](README.md).

Most of what follows can trap someone in crossed-wires, so if you plan to fiddle—and you should fiddle!—consider reading through this first.

---

## Quirks to Know

### **Karabiner runs the copy you paste in, not the file you pasted it from.** 

To change the rules, update that entry, either by using the edit button, or delete it completely and add the rule fresh.
Nothing you do to the JSON file afterwards will reach Karabiner on its own.

### **Verify what is actually loaded** before assuming any error. 

Run the below in Terminal to see a list of complex-modification manipulators in the *selected profile* only, not Simple Modifications, device settings, or other profiles.

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

### **Order matters; first match wins.** 

A manipulator carrying `optional: ["any"]` will match regardless of any extra modifiers, so placing one above a more specific rule makes the specific rule unreachable. A manipulator with no modifiers block matches bare presses only and will not shadow a modifier variant. 


### **The padlock key needs a `simultaneous` rule, not a modifier rule.** 

#### Why a `mandatory` modifier won't work:

| Press | Raw event order | Gap |
|---|---|---|
| Padlock key | `l` **first**, then `left_command` | same millisecond |
| Human typing `Cmd+L` | `left_option` first, then `l` | ~200 ms |

Two independent problems fall out of that:

1. The padlock emits its key code **before** its modifier, so when `l` is evaluated no
   modifier flag is set yet and `mandatory: ["left_command"]` cannot match.
2. On this Windows-layout board the key in the **`Cmd` position physically sends `Alt`**. A
   typed `Cmd+L` arrives as `left_option`, the swap rule converts it to `left_command` on
   the modifier's own key-down, and by the time `l` arrives the state genuinely *is* `Cmd`,
   so a `left_command`+`l` rule fires on every real `Cmd+L`.

Both presses look identical to any modifier-based rule, reached from opposite directions.
What separates them is **timing**, so the rule matches on that instead:

```json
"from": {
  "simultaneous": [ { "key_code": "l" }, { "key_code": "left_command" } ],
  "simultaneous_options": { "key_down_order": "insensitive", "key_up_order": "insensitive" },
  "modifiers": { "optional": ["any"] }
},
"parameters": { "basic.simultaneous_threshold_milliseconds": 20 }
```

The 20 ms threshold is what separates the two: the padlock's key and modifier arrive in the same millisecond, while a human pressing `Cmd`+`L` leaves ~200 ms between them. `key_down_order` is left insensitive — strict was tried and failed, and since both events share a millisecond there is likely no resolvable order for it to check. Cost: a plain `l` keystroke is buffered up to 20 ms while Karabiner waits for a possible modifier. Drop the threshold to 10 if that is ever perceptible.


### **Complex modifications are NOT device-scoped by default.** 
Unlike Simple Modifications (which have a target-device dropdown), a complex rule applies to *every* attached keyboard unless the manipulator carries a `device_if`/`device_unless` condition. This is why every manipulator in this rule set carries an `is_built_in_keyboard` guard.

### **Documented modification order from Karabiner's event-modification-chaining page:**
`1. Catch events from hardware → 2. Apply Simple Modifications → 3. Apply Complex Modifications`.


### **Manipulators do not re-process their own output.** 

A paired bidirectional swap
(`left_command→left_option` *and* `left_option→left_command` in one list) is safe and
does not cancel itself. 

---

## Diagnostic method

### **Karabiner-EventViewer answers one question well: "what does this key actually send?"**
It is not a good place to test whether a rule works and it's important to know why.

#### **There are two capture modes, and they show different layers:**
- **Capture Input Events**: events after macOS and Karabiner have processed them.
- **Capture Raw Input Events**, with the keyboard chosen in the device filter (8BitDo Retro 108 Keyboard (BitDo) [VID: …, PID: …]): what the board emits before anything touches it.

A rule's `from` block matches the raw layer, so raw is the view to write rules against. Designing from the processed view means matching against something that has already been transformed.

#### **Two consequences follow, and they cause different problems.**


* Raw capture bypasses your own rules, including for the app you're looking at. While raw capture is running, the frontmost EventViewer window receives the unmodified chord. A working rule can look broken. 

  * Example: `F4` sends `Cmd`+`Opt`+`M`; pressing it while EventViewer is in capture mode means EventViewer gets minimized, because macOS is getting the raw Cmd+Opt+M. 
    
#### **Never judge whether a rule fired from inside EventViewer. Use a real app, or the isolation test below.**
* The processed view lies about the source. 
  - Example: `F1` reads as a true `f1` in raw capture, but as `display_brightness_decrement` in Capture Input Events. The board sends a real `F1`; macOS converts it downstream. That conversion is controlled by System Settings > Keyboard > Keyboard Shortcuts > Function Keys ("Use F1, F2, etc. keys as standard function keys"). On some macOS versions that panel has no per-device dropdown and the setting applies to every keyboard, including the built-in one. It is a different panel from Modifier Keys and has nothing to do with modifier swaps.

#### **Reading the columns.** 

The **name** column shows *key codes*; the other columns show the *character produced*, and characters are not valid key codes. These are not the same thing, and confusing them is easy!


### **The isolation test** splits a broken rule in one keypress. 

Replace the rule's `to` block with something unmistakable:
```json
"to": [ { "shell_command": "open -a Calculator" } ]
```

* If nothing happens then `from` doesn't match.
* If the calculator opens, then the `from` block matches and the problem is in the `to` block.  

---

## Super Buttons — onboard, no software

Firmware-level, so fully functional on macOS.

### **To assign:** 

Press **`★`** (Fast key mapping) → **type the combo** (up to 6 keys simultaneously) → **press the Super Button**. That's combo first, super button second. You'll have plenty of time.

* **Cancel one mapping**: press `★`, then that Super Button.
* **Clear all**: hold `★` for 5 seconds.
* **Mapping mode auto-exits** after 60 seconds idle.
* **Key mapping indicator** blinks rapidly when a mapped button is pressed.
* **Stored in the keyboard**; survives reboots and follows it to any machine.
* **Factory reset** (`Pair` + `★` + `Profile`, 5 sec) wipes mappings, pairings and swaps.

### **Two Super Buttons are ON the board**

If no keys have been swapped yet, they're the two keys labelled **'B'** and **A**, sitting to the right of the spacebar, the middle two of the four-key group.
* They are firmware controls: they send nothing over USB, so Karabiner and EventViewer cannot see them at all. 
* That is expected, not a fault.

### **3.5mm jacks** 

The four **3.5mm jacks** on the back of a 108 model are labeled A, B, X, and Y. **They are not audio jacks.** 
The manual says "Interfaces for 8BitDo Super Buttons are for 8BitDo Keyboard Extensions only." That means these can be used for *additional*, external Super Buttons. They are set in the same manner as the keyboard version.
Note: The 108 comes with one set of external Super Buttons. Despite also being labeled **B** and **A**, they will be recognized as different Super Buttons from the keyboard.

**Two to ten, depending on what is plugged in.** The on-board pair is always there, so two
is the floor — nothing external required, and nothing to lose. Super Buttons are sold as
single-button and two-button units, so the four jacks carry between nothing and eight more,
putting the maximum at **ten**. All of it is in addition to the handful of unused keyboard
keys that can be customized.

### **To reach A/B from Karabiner**

Use the chord trick: record an unused combination, e.g. (`Ctrl`+`Alt`+`Shift`+`F9`) onto a Super Button with Fast Key Mapping, then catch that chord in a rule. The button then triggers anything a manipulator can express, including a `shell_command`.

### **How much this actually buys**

Rough arithmetic, for scale rather than precision. The twenty-two rules cover eleven keys
whose output is a matter of taste; with a hold on each that is twenty-one triggers the
picker can address today. Six to ten Super Buttons add that many again — discrete, labelled,
and reachable without spending any modifier state. Fanning every trigger out across the
modifier combinations Karabiner can distinguish pushes the theoretical ceiling into the
thousands.

**A caveat on those numbers.** They have not been independently checked, and the modifier
count rests on judgement calls — whether side-specific names count as distinct triggers,
whether `fn` belongs in the total, how many `simultaneous` members are realistic. Read them
as an order of magnitude and assume some of the detail is wrong. What is not in doubt is the
shape of the answer: the ceiling sits far past anything a person could remember, and further
past anything a person needs.

Which is why the Super Buttons are the interesting part and the big numbers are not. Ten
labelled physical switches cost nothing to recall — location does the remembering. A
thousand modifier states charge recall on every single use, and nothing on the desk tells
you they exist.

### **What to record onto a Super Button**

Fast Key Mapping records **physical keypresses**, so nothing a Super Button emits is unique
to it — the keys that recorded the combination can always send it again. Uniqueness has to
come from a combination nobody presses by accident, and that is a structural requirement
rather than caution.

Which combination matters more than it looks, because **the recorded chord competes with
this rule set's own modifier swaps.** The swaps source `left_command`, `left_option`,
`right_option` and `right_control`. A rule catching a chord built from any of those is
order-dependent against the swap manipulators: placed above them it sees pre-swap flags,
placed below them post-swap. That is a genuinely confusing bug to chase.

The four physical modifiers this rule set never touches are `left_shift`, `right_shift`,
`left_control` and `right_command`. Build the recording out of those:

| Recording | Verdict |
|---|---|
| `left_shift`+`right_shift`+`F9` | **Recommended.** Nothing in this rule set swaps either Shift, macOS claims neither, and nobody presses both at once by accident |
| `Ctrl`+`Alt`+`Shift`+`F9` | Works, but `Alt` is a swap source, so the catching rule's position relative to the swap manipulators changes what it sees |

**The reason is correctness, not headroom.** How many modifier states a recording leaves
spare is irrelevant — nobody uses more than a handful, so trading thirty-two theoretical
states for sixty-four buys nothing. What matters is that a chord built from a swapped
modifier is order-dependent against the swap rules, which is a bug that appears to come and
go. Pick unswapped modifiers and the question never arises.

**This works — tested, not inferred.** The `★` key programs a Super Button: press `★`, type
the combination, press the button. `★` is not involved in firing it afterwards. A programmed
Super Button emits its recorded combination exactly as the keyboard would, and
`Karabiner-EventViewer` shows the keystrokes, so a rule can catch them. That was the
premise the whole approach rested on and it holds.

An unmapped Super Button, by contrast, sends nothing at all — which is why the on-board pair
is invisible until you record something onto it.

Three further questions, two of them now settled by capture.

**A modifier held at press time does reach the rule.** Holding `left_control` while pressing
a Super Button programmed to `left_shift`+`right_shift`+`F9` produced an `f9` key-down
carrying `left_control, left_shift, right_shift` in its flags. So a Super Button can take
mandatory modifiers on top of its recorded combination, and the modifier fan-out is real
rather than assumed. Use an unswapped modifier, for the reason given above.

**A Super Button sustains its key-down; it does not pulse.** Four presses of the same button
held the `f9` key-down for 109 ms, 1,683 ms, 3,521 ms and 630 ms — tracking how long the
button was actually held. `to_if_held_down` therefore works on these buttons, and the
indicator blinking on each press means nothing about the report duration.

**The emission order is not stable**, which is worth knowing before building anything on it.
Across four presses of one button the two shifts arrived `left`→`right` twice and
`right`→`left` once. This is harmless for an ordinary rule, because modifiers are matched as
flags rather than as a sequence — but any `simultaneous` rule involving a Super Button must
set `key_down_order: insensitive`, exactly as the padlock rule does.

**Two Super Buttons can be chorded, and there is room to spare.** With one button recorded
to `left_shift`+`right_shift`+`F9` and a second to `left_shift`+`right_shift`+`F10`, pressing
both produced `f10` down, then `f9` down **18 ms later while `f10` was still held**, both keys
held together for 236 ms, and both released in the same millisecond. Overlapping, not
sequential — so a `simultaneous` rule catches them.

The shared modifiers are emitted once rather than twice, which makes the rule tidier than
expected: the two shifts go in `mandatory`, and only the two terminal keys go in
`simultaneous`.

```json
{
  "description": "Super Buttons A + B -> Mission Control",
  "from": {
    "simultaneous": [{ "key_code": "f9" }, { "key_code": "f10" }],
    "simultaneous_options": {
      "key_down_order": "insensitive",
      "key_up_order": "insensitive"
    },
    "modifiers": { "mandatory": ["left_shift", "right_shift"] }
  },
  "parameters": { "basic.simultaneous_threshold_milliseconds": 50 },
  "to": [{ "key_code": "mission_control" }],
  "type": "basic",
  "conditions": [
    {
      "type": "device_unless",
      "identifiers": [{ "is_built_in_keyboard": true }]
    }
  ]
}
```

**Use 50 ms, not the padlock rule's 20.** An 18 ms gap clears a 20 ms threshold by two
milliseconds, which is not a margin — one slower press and the chord silently stops firing.
The padlock key can afford 20 ms because both of its keys come from a single physical
switch; two Super Buttons are two switches and two reports.

`key_down_order` must be `insensitive` because the terminal keys arrive in whichever order
the buttons were pressed, and `key_up_order` likewise, since both released in the same
millisecond.

With that settled, every question about Super Buttons is answered: they are catchable, they
accept modifiers on top of their recording, they sustain a hold, and they chord with each
other. The only thing they cannot do is emit a keycode the keyboard itself could not send.

### **Do not use Fast Key Swap.** 

Holding two modifiers plus `★` swaps them at firmware level; the eligible keys are `Ctrl`, `Win`, `Alt` and `Shift`. E.g. `Win`+`Alt`+`★` swaps `Win` and `Alt`. This rule set already handles the modifier swap in Karabiner; doing both would apply it twice and cancel out. Firmware swaps are stored in the keyboard and will follow the board to other machines. None of that will show up in EventViewer. If you suspect issues, consider a factory reset.

### **`F13`+** 

Fast Key Mapping records physical keypresses, and this board has no keys above `F12`, so `F13` and up can't be recorded directly. Instead record an absurd chord (`Ctrl`+`Alt`+`Shift`+`F9`) onto a Super Button, then catch that chord in Karabiner and map it to anything, including a `shell_command`.

---

## Ultimate Software V2 — not available for this board on macOS

Retro 108 support landed in the **Windows v1.10** build. The macOS build's keyboard list covers only the Retro 87 series and Retro 68-N40. So the 3 custom profiles, the macro editor, and firmware updates all require a Windows machine or a VM with USB passthrough.
* The **Profile button** is effectively inert without that software; whether it is *completely* inert is unverified. It will still work for the factory-reset combo, but nothing more. Profiles and onboard key mappings are stored separately.

**Firmware updates aren't critical.** 8BitDo's keyboard-side firmware changelog is Windows quality-of-life work ("optimized macro stability", language/usage fixes) on features unused here. Revisit only if Bluetooth or 2.4G starts misbehaving; even then you'll probably need a Windows instance to update it.

---

## Physical layout notes

**Scroll Lock and Pause land on brightness, by design not accident.** These specific rules map them
to `f14` / `f15`, and macOS treats **`F14` / `F15` as the legacy brightness-down / -up keys**.
The board genuinely sends `scroll_lock` and `pause`; Karabiner converts; macOS interprets.
Left as-is deliberately, it's free brightness control on a board that has none. Need yet MORE customizable buttons? These are fair game.

Top-right cluster past `F12`: `FN`, `Mute`, `Calculator`, `Screen lock`. Three
keys sit between `F12` and `FN` (`PrtScrn` / `ScrollLock` / `Pause`).

Bottom row, right of the spacebar: the two **A** / **B** Super Buttons, then
`right_control` as the farthest-right key. `A` and `B` register nothing in EventViewer
because they are firmware controls, and they work correctly on a Mac as shipped. 
`right_control` is currently consumed by the `right_control -> right_option` swap.

__Works natively on macOS with no rules:__ tri-mode connectivity (USB-C / 2.4G dongle /
Bluetooth), volume knob, Mute key, `Fn+F9`/`F10`/`F11` as the printed media keys, N-key rollover,
hot-swap sockets.

Mode switch positions: `OFF` = wired, `2.4` = dongle, `BT` = Bluetooth.
Battery 2000mAh, ~200h. Auto-shutdown after 15 min idle (not while wired).

---

### Other colourways

The board ships in more than one colourway. One variant is reported to carry graphics on the
keycaps with additional legends, apparently Japanese, and to come without the A/B Super
Button module. *Unverified* — reported rather than examined.

Nothing in this rule set depends on it either way. **A keycap legend does not change what a
key sends**, so a differently printed board produces identical key codes and every rule here
applies unchanged. The only difference that would matter is a missing physical key, and the
`README` explains how to drop the rules for a key you do not have.

## Still open (as configured here)

- `Fn`+number and `Fn`+letter combinations were never swept. Likely empty — the board has a dedicated Pair button, so Bluetooth slots are probably not on the number row. Worth a look only if you need more free chords to map.

---

## Sources

- Manual (image-only PDF, 60pp, English on pages 1–7, seven additional languages included): <https://download.8bitdo.com/Manual/PC-Peripherals/Retro-108-Mechanical-Keyboard.pdf>
- Ultimate Software V2 release notes / device matrix: <https://app.8bitdo.com/Ultimate-Software-V2/>
- Official Karabiner rule repo: <https://github.com/pqrs-org/KE-complex_modifications>