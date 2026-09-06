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
Note: The 108 comes with one set of external Super Buttons. Despite also being labeled **B** and **A**, they will be recognized as different Super Buttons from the keyboard. With four external Super Buttons, that creates ten Super Button options. That's in addition to the handful of unused keyboard keys that can be customized.

### **To reach A/B from Karabiner**

Use the chord trick: record an unused combination, e.g. (`Ctrl`+`Alt`+`Shift`+`F9`) onto a Super Button with Fast Key Mapping, then catch that chord in a rule. The button then triggers anything a manipulator can express, including a `shell_command`.

### **Do not use Fast Key Swap.** 

Holding two modifiers plus `★` swaps them at firmware level; the eligible keys are `Ctrl`, `Win`, `Alt` and `Shift`. E.g. `Win`+`Alt`+`★` swaps `Win` and `Alt`. This rule set already handles the modifier swap in Karabiner; doing both would apply it twice and cancel out. Firmware swaps are stored in the keyboard and will follow the board to other machines. None of that will show up in EventViewer. If you suspect issues, consider a factory reset.

### **`F13`+** 

Fast Key Mapping records physical keypresses, and this board has no keys above `F12`, so `F13` and up can't be recorded directly. Instead record an absurd chord (`Ctrl`+`Alt`+`Shift`+`F9`) onto a Super Button, then catch that chord in Karabiner and map it to anything, including a `shell_command`.

---

## Ultimate Software V2 — not available for this board on macOS

Retro 108 support landed in the **Windows v1.10** build. The macOS build's keyboard list covers only the Retro 87 series and Retro 68-N40. So the 3 custom profiles, the macro editor, and firmware updates all require a Windows machine or a VM with USB passthrough.
* The **Profile button** is effectively inert without that software (I'm exploring whether this is completely true. Stay tuned.). It will still work for the factory-reset combo, but nothing more. Profiles and onboard key mappings are stored separately.

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

## Still open (as configured here)

- `Fn`+number and `Fn`+letter combinations were never swept. Likely empty — the board has a dedicated Pair button, so Bluetooth slots are probably not on the number row. Worth a look only if you need more free chords to map.

---

## Sources

- Manual (image-only PDF, 60pp, English on pages 1–7, seven additional languages included): <https://download.8bitdo.com/Manual/PC-Peripherals/Retro-108-Mechanical-Keyboard.pdf>
- Ultimate Software V2 release notes / device matrix: <https://app.8bitdo.com/Ultimate-Software-V2/>
- Official Karabiner rule repo: <https://github.com/pqrs-org/KE-complex_modifications>