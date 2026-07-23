# Fixes You Need After Buying a Sovol SV08

> Fixes you need after buying a Sovol SV08: warped bed (v1 & v2 — no DIY fix works), paused prints going cold and ruining the job, timelapse autorender resetting every boot, the Moonraker "too old" nag, and cryptic G-code macro buttons.

Twelve things on a stock **Sovol SV08** that you will hit sooner or later — plus one worth adding:

1. **Timelapse auto-render turns itself off after every reboot.**
2. **Mainsail nags that Moonraker is too old** — and the updater is disabled.
3. **The macro dashboard is full of cryptic buttons** named `G31`, `G34`, `M106`, `M600`.
4. **A paused print goes cold after 30 minutes and is ruined.** ← this one costs you real prints
5. **The bed is warped** — on both the original bed and the "v2" revision. ← this one costs you *every* print
6. **Random `Timer too close` shutdowns mid-print.** Every forum will tell you it's heat or EMI. On my machine it was the **stock 8 GB eMMC** — swapping it fixed what nothing else would. ← this one costs you the board
7. **Power-loss recovery calls `os.fsync()` on every single move.** It kills prints with organic supports or z-hop *now*, and it is what makes a slow or worn eMMC (#6) fatal. **Do this one first — it's free.** ← this one costs you both
8. **`M106 P1` doesn't address one fan — it sets both.** The `P` parameter is decorative, one branch of the macro is dead code, and the five fans on the machine are documented nowhere. ← this one costs you an afternoon
9. **Your exhaust fan is PID-controlled by a CPU temperature it cannot possibly affect** — so it runs at **100%, permanently, forever**. If you have an enclosure, it is venting your chamber around the clock and you will never hold a temperature in it. Meanwhile the fan that *can* cool that CPU is PID'd to a target it has already reached, so it never spins up. **Both mainboard fans are wrong, in opposite directions.** ← this one costs you every ABS print
10. **The filament sensor pauses your print on a single read of a switch.** No debounce, anywhere. One bounce of a mechanical lever on a machine that shakes — and the machine that shakes it is the one running your job — parks the toolhead mid-print with the filament still loaded. ← this one costs you prints, at random, for no reason
11. **Your printer beacons to a cloud service you never signed up for**, every two seconds, forever — broadcasting its local IP and hostname to `app.obico.io`, unlinked, with crash telemetry opted in on your behalf. ← this one costs you privacy
12. **The SSH host keys were generated at the factory in December 2023 and shipped to every unit** — and the default user has passwordless `sudo`. ← this one costs you the machine
13. **There's no one-button way to park the head for maintenance.** You hand-jog three axes every time you want to reach the nozzle or wipe the bed — and jog Y the wrong way and you crash into the back frame. ← the one thing here to *add*, not fix
14. **`G28` homes Y into the one corner where the filament tube gets pinched — and sensorless homing false-triggers on the drag.** Sovol homes both X and Y positive, into the right-rear corner. The tube binds between toolhead and frame there, StallGuard reads the drag as a stall, and Y stops a few mm early. Only bites with the gantry high — roughly mid Z travel and up, where the tube is pulled taut; homing from a low gantry looks clean. Sovol's answer is `set_position_z: 0` plus a blind `G0 Z5`, which throws away a known-good Z to lift over the problem. Home X to the *left* instead and the corner never happens. ← costs you a few mm of Y, silently, and chews the tube

The first three are annoyances. **[Fix 4](#fix-4--a-paused-print-goes-cold-after-30-minutes-and-is-ruined) will destroy a 20-hour job** — pause for a filament change, get distracted, come back to a cold printer and a part that has let go of the bed. **[Fix 5](#fix-5--the-bed-is-warped--and-no-diy-fix-actually-works) is the one that quietly taxes every single print you make**, and it is the only one here whose real answer is "buy a new part".

**[Fix 6](#fix-6--timer-too-close-is-the-emmc-not-a-hot-mainboard) and [Fix 7](#fix-7--power-loss-recovery-writes-to-flash-on-every-single-move) are the same story told twice.** Sovol's power-loss-recovery patch forces a write to physical flash on every move; that stalls the host until the MCU gives up (`move queue overflow`, `Timer too close`), and it grinds the eMMC to death over months. I chased the `Timer too close` shutdowns through every forum thread I could find — heat, EMI, fans — and none of it worked. Replacing the eMMC did. Fix 7 is why it wore out.

Each fix below covers what causes it, how to fix it, how to verify it, and how to revert — including *why the obvious fix is wrong* in most of them.

**[Fix 12](#fix-12--ssh-host-keys-from-the-factory-and-passwordless-root) is the one to do today.** It is free, it takes one minute, it cannot break anything, and until you do it your printer is trivially impersonable by anyone who has downloaded Sovol's own disk image.

Tested on an SV08 running the factory Sovol image (Klipper + Moonraker + Mainsail, Moonraker `v0.8.0-209-g4235789-dirty`). Fixes 1–4 and 7–12 are software and cost nothing. **Fixes 5 and 6 are hardware and cost money** — be slow to accept those two, and do the free ones first: [Fix 7](#fix-7--power-loss-recovery-writes-to-flash-on-every-single-move) in particular may be the whole of your [Fix 6](#fix-6--timer-too-close-is-the-emmc-not-a-hot-mainboard) problem.

**Own an [SV08 Max](#does-any-of-this-apply-to-the-sovol-sv08-max) instead?** Sovol quietly fixed 4, 9 and 10 on it — and shipped Fix 7 forward with the guard *loosened*. Read that section before you apply anything here.

Most of this isn't even SV08-exclusive: the timelapse bug hits any `moonraker-timelapse` install, the macro-button problem hits any Klipper printer whose vendor named macros after raw G-code, and the paused-print cool-down hits any Klipper machine with a stock `idle_timeout`. The warped bed is the SV08-specific one.

> **Back up before you touch anything.** `cp file file.bak.$(date +%Y%m%d)`. Every fix below tells you how to revert it.

| # | Problem | Cost | Severity |
|---|---|---|---|
| [1](#fix-1--timelapse-auto-render-resets-to-off-after-every-reboot) | Timelapse auto-render resets to OFF on every reboot | Free | Annoyance |
| [2](#fix-2--mainsail-moonraker-too-old-update-to-at-least-v080-306) | Mainsail: "Moonraker too old" + updater disabled | Free | Cosmetic |
| [3](#fix-3--mainsail-dashboard-is-full-of-g31--g34--m106--m600-buttons) | Dashboard full of raw G-code buttons | Free | Annoyance |
| [4](#fix-4--a-paused-print-goes-cold-after-30-minutes-and-is-ruined) | Paused print cools down and is ruined | Free | **Destroys prints** |
| [5](#fix-5--the-bed-is-warped--and-no-diy-fix-actually-works) | Warped bed (v1 **and** v2) — no DIY fix works | 💸 New bed | **Degrades every print** |
| [6](#fix-6--timer-too-close-is-the-emmc-not-a-hot-mainboard) | `Timer too close` shutdowns — it's the eMMC, not heat | 💸 New eMMC | **Destroys prints + the board** |
| [7](#fix-7--power-loss-recovery-writes-to-flash-on-every-single-move) | PLR `os.fsync()` per move → `move queue overflow`, dead flash | Free | **Destroys prints + your eMMC** |
| [8](#fix-8--m106-p1-does-not-do-what-you-think-and-nobody-documents-the-fans) | `M106 P` is decorative — both part fans always move together | Free | Undocumented |
| [9](#fix-9--your-exhaust-fan-is-pid-controlled-by-a-cpu-it-isnt-pointed-at) | Both mainboard fans PID'd to the wrong thing — exhaust stuck at 100%, board fan stuck at 10% | Free | **Wrecks enclosed printing** |
| [10](#fix-10--one-bounce-of-the-filament-switch-pauses-your-print) | Filament sensor pauses on a single switch read — no debounce | Free | **Pauses prints at random** |
| [11](#fix-11--your-printer-announces-itself-to-a-cloud-service-you-never-signed-up-for) | Unlinked Obico beacons your LAN IP to a cloud, every 2 s, forever | Free | **Privacy + 20 MB of junk on your eMMC** |
| [12](#fix-12--ssh-host-keys-from-the-factory-and-passwordless-root) | Factory SSH host keys on every unit + passwordless root | Free | 🔴 **Anyone on your LAN owns the printer** |
| [13](#fix-13--theres-no-one-button-way-to-park-the-head-for-maintenance) | No one-button park-for-maintenance position — hand-jog every time | Free | ➕ Add-on, not a defect |
| [14](#fix-14--g28-homes-y-into-the-corner-that-pinches-the-filament-tube) | `G28` homes X and Y into the right-rear corner, pinching the filament tube | Free | **Damages the tube; sensorless Y false-triggers a few mm early — at high Z (mid travel and up)**. ⚠️ Fix unproven — test with a hand on the power button |

---

## Fix 1 — Timelapse auto-render resets to OFF after every reboot

### Symptoms

* You enable **Timelapse → Auto render** in Mainsail. It works.
* You reboot the printer. It's off again.
* The toggle in the UI may appear **greyed out / locked**.
* Prints finish, frames are captured, but **no video is ever produced**.

### Cause

Sovol's factory `moonraker.conf` ships this line in the `[timelapse]` section:

```ini
[timelapse]
output_path: ~/printer_data/timelapse/
frame_path: /tmp/timelapse/printer
autorender: False        # <-- this
```

`moonraker-timelapse` initialises in this order (`~/moonraker-timelapse/component/timelapse.py`):

1. Load built-in defaults — `autorender: True`.
2. Load your saved settings from Moonraker's LMDB database. **This is where the Mainsail toggle persists.**
3. Call `overwriteDbconfigWithConfighelper()` — **any key present in `moonraker.conf` overwrites the database value**, and the key is appended to `blockedsettings`.

Step 3 stomps your saved `True` back to `False` on every single boot. The UI toggle only ever worked for the current session. And because the key lands in `blockedsettings`, the frontend greys the control out.

### Fix

Comment the line out. Don't set it to `True` — **remove it from the config entirely**, or you've just moved the problem.

```bash
ssh sovol@<printer-ip>
cd ~/printer_data/config
cp moonraker.conf moonraker.conf.bak.$(date +%Y%m%d)
nano moonraker.conf
```

```ini
[timelapse]
output_path: ~/printer_data/timelapse/
frame_path: /tmp/timelapse/printer
#autorender: False       # disabled - was clobbering the DB setting on every boot
```

```bash
sudo systemctl restart moonraker
```

### Verify

```bash
curl -s http://<printer-ip>:7125/machine/timelapse/settings | python3 -m json.tool
```

You want:

```json
"autorender": true,
"blockedsettings": []
```

**`blockedsettings: []` is the real proof.** It means `moonraker.conf` is no longer overriding the database, so the Mainsail toggle now persists across reboots — set it either way and it will stick.

### The general trap

This bites **every** `moonraker-timelapse` setting, not just `autorender`. Any key you put in `moonraker.conf` silently wins over the frontend and greys the control out. **If you want a timelapse setting to be UI-controllable, it must not appear in `moonraker.conf` at all.**

### Revert

Uncomment the line, restart Moonraker.

---

## Fix 2 — Mainsail: "Moonraker too old, update to at least v0.8.0-306"

### Symptoms

Mainsail shows a persistent notification:

> **Dependency: Moonraker.** Current version does not support all features of Mainsail. Update Moonraker to at least **v0.8.0-306**.

And the one-click updater doesn't work — the **Update Manager / Machine page is empty or 404s**.

### Cause

Two separate things:

1. Sovol ships **Moonraker `v0.8.0-209`** — about 97 commits behind what Mainsail wants. The warning is a **cosmetic frontend compatibility check**. It is *not* a print fault. Nothing is broken; Mainsail is just telling you some of its newer features have no backend to talk to.
2. `machine/update/status` returns **404** — Sovol **disabled the `update_manager` component**. So there is no one-click update button to press.

And here's the part that matters:

```bash
cd ~/moonraker && git describe --tags --long --dirty
v0.8.0-209-g4235789-dirty
```

That **`-dirty`** suffix means **Sovol shipped a modified Moonraker**. A plain `git pull` will hit merge conflicts, or silently wipe the Sovol-specific patches that make the SV08's hardware work. **Do not blindly `git pull` Moonraker on an SV08.**

### Your three options

| Option | What happens |
|---|---|
| **Ignore it** | Genuinely fine. It is cosmetic. Everything prints. |
| **Actually update Moonraker** | The *correct* fix, but you must reconcile Sovol's `-dirty` changes by hand. Real risk of bricking the web stack. Only worth it if you need a specific new Mainsail feature. |
| **Silence the notification** | Below. Zero risk to printing, but it is a **workaround, not an update** — you are telling Mainsail a version number that isn't true. |

### Fix (silence it)

**Be honest with yourself about what this is:** it makes Moonraker *report* a version it does not have. Mainsail will then stop warning you and will happily enable UI features whose backend endpoints don't exist. Those specific features will fail. In practice, for normal printing, nothing does.

Only do this if you have decided you are not going to update. **Never restart Moonraker mid-print.**

```bash
ssh sovol@<printer-ip>
cd ~/moonraker/moonraker/utils
cp __init__.py __init__.py.bak.$(date +%Y%m%d)
nano __init__.py
```

Find `get_software_info()` (~line 139) and add **one line** as the new first statement:

```python
def get_software_info() -> Dict[str, Any]:
    return {"software_version": "v0.8.0-400"}      # <-- add this line
    src_path = source_info.source_path()
    if source_info.is_git_repo():
        return get_repo_info(str(src_path))
    ...
```

The early `return` short-circuits the rest. `v0.8.0-400` is simply any value ≥ Mainsail's required `v0.8.0-306`.

```bash
sudo systemctl restart moonraker
sleep 10     # the API needs ~8s before it answers again - don't panic at 6s
```

### Verify

```bash
curl -s http://<printer-ip>:7125/server/info | python3 -m json.tool
```

```json
"moonraker_version": "v0.8.0-400",
"klippy_state": "ready",
"warnings": []
```

Hard-refresh the Mainsail tab and the notification is gone.

### Why patch the code instead of dropping in a `moonraker/.version` file?

Because it won't work here. Moonraker only reads `.version` on the **non-git** code path. `~/moonraker` **is** a git repo, so version resolution goes through `git describe` and the file is ignored entirely. Patching `get_software_info()` is the only thing that intercepts it.

### Revert

```bash
cp ~/moonraker/moonraker/utils/__init__.py.bak.<date> ~/moonraker/moonraker/utils/__init__.py
sudo systemctl restart moonraker
```

---

## Fix 3 — Mainsail dashboard is full of `G31` / `G34` / `M106` / `M600` buttons

### Symptoms

The Mainsail macro panel shows ~32 buttons, and about seven of them are named after **raw G-code commands** — `G31`, `G34`, `M106`, `M107`, `M109`, `M190`, `M600` — with no hint what they do.

### Cause

Mainsail's macro panel defaults to `mode: simple`, which renders **every macro that doesn't start with `_`** as a dashboard button. Sovol defined these seven as normal macros, so they all show up.

### The mistake to avoid

**Do not rename them.** These names are not cosmetic — they are the *interface*:

* Your **slicer** emits `M106` / `M107` / `M109` / `M190` / `M600` directly in the G-code.
* The **power-loss-recovery scripts** call `G31`.

Rename them in place and you break printing. On an SV08 they map to:

| Macro | Where | What it does |
|---|---|---|
| `G31` | `plr.cfg` | Clear power-loss-recovery state |
| `G34` | `Macro.cfg` | Bed mesh clear → home → quad gantry level → home Z → park |
| `M106` / `M107` | `Macro.cfg` | Part cooling fan on (no `S` = 100%) / off |
| `M109` / `M190` | — | Wait for hotend / bed temperature |
| `M600` | `Macro.cfg` | Filament change (`PAUSE STATE=filament_change`) |

### Fix — wrap, then hide

Keep every original. Add **friendly-named wrapper macros** that call them, then hide the raw-named ones from the UI.

**1. Install [`aliases.cfg`](aliases.cfg)** into `~/printer_data/config/`:

```ini
[gcode_macro LEVEL_GANTRY]
description: Home + quad gantry level + park (was G34)
gcode:
    G34

[gcode_macro CHANGE_FILAMENT]
description: Pause for filament change (was M600)
gcode:
    M600

[gcode_macro CLEAR_RESUME_FLAG]
description: Clear power-loss-recovery state (was G31)
gcode:
    G31

[gcode_macro FANS_MAX]
description: Part cooling fans 100%
gcode:
    M106 S255

[gcode_macro FANS_OFF]
description: Part cooling fans off
gcode:
    M107
```

**2. Include it** — add to `printer.cfg`, *after* the existing `[include Macro.cfg]`:

```ini
[include aliases.cfg]
```

**3. `FIRMWARE_RESTART`.**

**4. Hide the raw-named buttons.** You do *not* need Mainsail's "expert" mode for this — a plain `hiddenMacros` list is enough. One API call:

```bash
curl -X POST http://<printer-ip>:7125/server/database/item \
  -H "Content-Type: application/json" \
  -d '{"namespace":"mainsail","key":"macros","value":{
        "mode":"simple",
        "hiddenMacros":["G31","G34","M106","M107","M109","M190","M600"],
        "macrogroups":{}
      }}'
```

**Reload the browser tab** — Mainsail only reads this database at page load.

This is **UI-only**. Klipper still has every macro; your slicer and the PLR scripts are completely unaffected.

### Revert

```bash
curl -X POST http://<printer-ip>:7125/server/database/item \
  -H "Content-Type: application/json" \
  -d '{"namespace":"mainsail","key":"macros","value":{"mode":"simple"}}'
```

Then remove `[include aliases.cfg]` from `printer.cfg` and `FIRMWARE_RESTART`.

---

## Fix 4 — A paused print goes cold after 30 minutes and is ruined

This is the one that actually costs you money.

### Symptoms

* You trigger a filament change (`M600`), or pause to clear a clog, or swap colour.
* You get distracted for half an hour.
* You come back to a **stone-cold printer**. Hotend off, bed off, steppers released.
* Resume is hopeless: the part has **let go of the bed**, or the nozzle has **cold-locked** with filament in it. The print is gone — sometimes 20 hours in.

### Cause

Stock SV08 ships a **30-minute** idle timeout:

```ini
[idle_timeout]
timeout: 1800
```

Klipper's `idle_timeout` **does not care that you are paused**. It sees no movement, the timer expires, and it runs `TURN_OFF_HEATERS` + `M84`. Thirty minutes is not a lot of time when a filament change turns into "where did I put the other spool".

Mainsail actually ships a proper mechanism for this — `PAUSE` is supposed to call `SET_IDLE_TIMEOUT` to extend the timer while paused, and restore it on `RESUME`. **On the SV08 it is dead twice over:**

1. `mainsail.cfg:22` — `[gcode_macro _CLIENT_VARIABLE]` is **commented out**, so `variable_idle_timeout` defaults to `0`, which means "don't set anything". No-op.
2. `PAUSE` is **defined twice** — once at `mainsail.cfg:93` and again at `Macro.cfg:414`. `printer.cfg` includes `Macro.cfg` **last**, so Klipper merges the sections and **Sovol's `gcode:` body wins**. Mainsail's `SET_IDLE_TIMEOUT` pause logic never executes.

So on a stock SV08 there is nothing protecting a paused print. The 30-minute timer just runs.

### Fix

Two parts. Both are needed.

**1. Raise the timeout** in `printer.cfg`:

```ini
[idle_timeout]
gcode: _IDLE_TIMEOUT
timeout: 43200          # 12h, was 1800 (30 min)
```

**2. Make the timeout macro leave the heaters alone when paused.** Find `_IDLE_TIMEOUT` in `Macro.cfg` and branch on print state:

```ini
[gcode_macro _IDLE_TIMEOUT]
gcode:
    {% if printer.print_stats.state == "paused" %}
      RESPOND TYPE=echo MSG="Idle timeout reached, but print is paused - holding temperature."
    {% else %}
      M84
      TURN_OFF_HEATERS
    {% endif %}
```

`FIRMWARE_RESTART`.

### What this actually does — read this bit

The two parts do different jobs, and the second one is the important one:

* The **timeout value** governs *ordinary* idle — job finished, printer sitting there doing nothing. That's the `else` branch, and it still cools down normally. Raising it to 12h just means an idle printer takes longer to switch its heaters off.
* The **paused branch never cools down at all.** So a pause is **not** "protected for 12 hours" — it is **protected indefinitely**. The timer fires, hits the `paused` branch, prints a message, and does nothing. There is no window to miss.

That is the entire point. A pause that cools down on a timer is a pause that can silently destroy a 20-hour print. A pause that holds temperature can only ever waste electricity.

### ⚠️ The tradeoff — know what you are accepting

**A paused print will now hold the hotend and bed at full temperature forever, unattended, until someone resumes it or cuts the power.** On PETG or ABS that is a nozzle sitting at ~240–260 °C indefinitely.

That is a deliberate trade: **a wasted print is certain if it cools; a hazard is only possible if it stays hot.** But it is a real trade, and you should make it consciously:

* Do not walk away from the house with a print paused.
* `M600` and `PAUSE` are **one-click in Mainsail with no confirmation dialog** — a stray tap on the dashboard now leaves your printer hot until you notice. (Fix 3 above hides the raw `M600` button, which helps.)
* If you are not comfortable with that, use the **middle-ground variant** instead: keep the generous timeout but let it *eventually* cool, by deleting the `if/else` and always running `TURN_OFF_HEATERS`. You then get 12 hours of pause instead of infinite — enough for any realistic filament change, with a guaranteed cool-down at the end.

Pick whichever you can live with. Just don't leave it at the stock 30 minutes, because that is long enough to feel safe and short enough to ruin the print.

### Verify

```bash
curl -s "http://<printer-ip>:7125/printer/objects/query?idle_timeout&pause_resume&print_stats" | python3 -m json.tool
```

Then test it for real: start a print, `PAUSE`, and confirm the hotend target does not drop.

### Revert

Restore `timeout: 1800` in `printer.cfg`, restore the original `_IDLE_TIMEOUT` body, `FIRMWARE_RESTART`.

---

## Fix 5 — The bed is warped — and no DIY fix actually works

This is the only fix on this page that costs money, and it is the only one I found no software answer to. **I tried every DIY solution I could find online. None of them worked.** In the end I replaced the bed, and that was the only thing that actually fixed it.

### Symptoms

* First layer is **perfect in one region and wrong in another** — squished in the middle and barely touching at the edges, or the reverse.
* You re-level, re-mesh, re-tram. It helps for one print, then it's back.
* Large prints lift at the corners; small prints in the centre are fine, so you keep half-believing the printer is OK.
* **The mesh changes shape when the bed is hot.** Probe cold, probe at 60 °C, probe at 100 °C — you get three different surfaces.
* You buy the **v2 bed** hoping it's fixed. It is not. **Both the original SV08 bed and the v2 revision warp.**

### Cause

The stock plate is a thin aluminium sheet with the heater and magnetic layer laminated to it. It is not flat when it leaves the factory, and — worse — **it does not hold whatever shape it has**. The lamination and the aluminium expand at different rates, so the plate *changes shape as it heats*. That is thermal bow, not a levelling error.

This distinction is the whole reason the usual advice fails.

### Why the DIY fixes don't work — save yourself the weekend

I tried all of these. Here is what each one actually addresses, and why it isn't enough:

| What people tell you to do | Why it doesn't fix it |
|---|---|
| **More bed-mesh points / adaptive meshing (KAMP)** | A mesh only corrects a *static* height map. It cannot correct a surface that is a different shape at print temperature than it was when probed. And a mesh is a correction, not a repair — you are asking the Z axis to trace a wave for the whole print. |
| **Quad gantry level (QGL)** | QGL levels the **gantry** relative to the bed's corners. It makes the gantry parallel to a plane. **A warped bed is not a plane.** QGL cannot fix flatness; it was never meant to. |
| **Shim / re-torque the bed mounting screws** | Four mounting points can tilt a plate and can pull its corners. They cannot flatten a dome or a wave in the middle. You just move the error around. |
| **Heat the bed, then re-tighten the screws** | Sounds clever — bolt in the hot shape. But the bed still passes through every temperature on the way there, and you have now pre-stressed the plate. |
| **Thicker / different PEI spring steel sheet** | The spring steel is 0.5 mm and *flexible by design* — it conforms to whatever is under it. It cannot flatten the plate it sits on. A textured sheet only *hides* a bad first layer. |
| **Glass or a cast tooling plate on top** | Genuinely flat, and it does work — but you lose the magnetic sheet and flex-to-release, you add mass and thermal lag, and you have to re-do Z offset. It's a workaround with real costs, not a fix. |
| **Raise the first layer height / slow it down** | Papering over it. You are trading dimensional accuracy and adhesion for the appearance of a fix. |

The pattern: **every software fix compensates, and every mechanical fix tilts.** Neither of those is flatness. If the plate isn't flat and doesn't *stay* flat, there is nothing on the printer to fix.

### Fix — replace the bed

I replaced the stock bed with the **[R3MEN graphite heated bed for the Sovol SV08](https://www.r3men.com/products/graphite-heated-bed-for-sovol-sv08)**. It is a drop-in replacement for the SV08 bed. This is the only thing that solved it for me — first layer became consistent across the whole 350 × 350 plate, hot and cold, and stayed that way.

**Two honest caveats**, because I don't want to sell you anything:

* I am a customer, not affiliated. I have **not** independently measured the plate's flatness with a straightedge and feeler gauges — what I can report is the outcome, which is that the problem went away and has not come back.
* It is **not cheap**, and you are replacing a part on a printer you just bought. That is a genuinely annoying thing to have to accept. I accepted it only after the DIY route wasted more of my time than the bed cost.

Any bed that is actually flat and actually *stays* flat when hot will solve this. That one is what I used.

### Verify

Do this **hot**, and do it at more than one temperature — that is the whole point.

```bash
# Heat the bed and let it soak. 10+ minutes, not 2 - the plate keeps moving after the sensor says it's there.
# Then, in the Mainsail console:
BED_MESH_CLEAR
BED_MESH_CALIBRATE
```

Look at the mesh in Mainsail's **Heightmap** tab and read the **range** (max − min), not the pretty colours — the colour scale auto-fits, so a terrible bed and a great one can look equally dramatic.

* Repeat at **60 °C**, then again at **100 °C**.
* A good bed: a small range, and **roughly the same shape at both temperatures**.
* A warped bed: a large range that **changes shape between the two** — that is the thermal bow, and it is the thing a mesh cannot save you from.

Compare before and after. That is your evidence.

### Revert

Bolt the old bed back on and re-run `BED_MESH_CALIBRATE` + Z offset. Keep the stock bed — it is a fine spare, and you will want it if you ever RMA or sell the machine.

---

## Fix 6 — "Timer too close" is the eMMC, not a hot mainboard

### Symptoms

* Mid-print, usually **late in a long print**, everything stops:

  > **Klipper has shutdown. MCU 'extra_mcu' shutdown: Timer too close**

* Same message on the KlipperScreen LCD and in `klippy.log`.
* It's intermittent. Short prints are fine. The longer the job, the likelier it dies.
* It comes back after a `FIRMWARE_RESTART` — and then happens again.

### What the internet will tell you (and why I stopped believing it)

Search this error and you'll be told it's **heat** (put a fan on the mainboard) or **EMI** on the USB line to the toolhead (re-route your cables). The [Sovol forum's own thread on it](https://forum.sovol3d.com/t/sv08-overcoming-high-temperature-klipper-shutdown-problem-my-solution/7623) ends with someone cutting a hole in their desk and bolting a **160 CFM** fan under the machine — and the thread still argues about whether heat was ever the cause.

**On my machine it was the eMMC module.** I went through the forums and tried everything I could find — none of it worked. I swapped the stock eMMC for a 32 GB module off eBay and the shutdowns stopped. No fan, no cable re-routing, no config change.

> **What I can and can't tell you.** The stock SV08 ships an **8 GB eMMC**. I replaced it with a 32 GB one, so I *upgraded* as well as replaced — and I no longer have the original to autopsy. **I don't know whether the stock module was worn out or simply cheap and slow.** Both produce this failure, for the same underlying reason (below), and I'm not going to pretend I know which one it was. Sovol themselves now sell a [32 GB eMMC upgrade](https://www.sovol3d.com/products/sv08-max-32gb-emmc), which tells you something about the stock part.

> **Honesty about this one:** I did not capture `dmesg` before I swapped the module, so I can't hand you a smoking-gun log line from my own printer. What follows is the mechanism that fits, plus **the check I wish I'd run first** — so you can confirm it on *your* machine before spending money.

### Why a bad eMMC produces exactly this error

`Timer too close` is the **MCU** complaining that the **host** missed a deadline. Klipper's host process schedules moves ahead of time and streams them to the MCU; if the host stalls long enough, the MCU runs out of runway and shuts down to avoid doing something dangerous.

An eMMC stalls the host beautifully, and it doesn't have to be *broken* to do it — only **slow at synchronous writes**:

* **A worn module** starts retrying internally as flash degrades, and a write that normally takes microseconds blocks for **hundreds of milliseconds**.
* **A cheap, small module is slow even when perfectly healthy.** An 8 GB eMMC has few flash dies, so it has little internal parallelism and poor random-write latency — precisely the workload [Fix 7](#fix-7--power-loss-recovery-writes-to-flash-on-every-single-move) generates.

Either way klippy blocks inside that write, misses its scheduling window, and the MCU shuts down. It's a host-side stall wearing an MCU-side error message — which is exactly why chasing mainboard temperature never fixes it.

It also explains something heat doesn't:

* **Why it hits late in long prints.** More writes, more accumulated retries. (Though heat has the same shape — see below.)

And it explains a confounder worth knowing about: **pulling the machine apart to mount a fan means reseating the eMMC.** If a fan "fixed" it for you, you changed two things at once.

### Where the title of this section overstates it

The heading says "not a hot mainboard". That was true on **my** machine, and I'll stand behind the mechanism above — but I have since measured something that makes the flat dismissal of heat wrong, and I'd rather correct it than defend it.

`Timer too close` is a *host missed its deadline* error. **Anything** that stalls the host produces it. There are at least three plausible causes on this platform, and they are not mutually exclusive:

| Cause | Evidence | Cost to check |
|---|---|---|
| **eMMC stall** (slow or worn synchronous writes) | This section. Fixed it on my printer. | **Free** — [sysfs check](#check-yours-before-you-buy-anything), then [Fix 7](#fix-7--power-loss-recovery-writes-to-flash-on-every-single-move) |
| **Heat** | Real, and I under-rated it. See below. | Free to *measure* |
| **EMI on the toolhead USB line** (PSU switching noise) | The second theory in the [Sovol forum thread](https://forum.sovol3d.com/t/sv08-overcoming-high-temperature-klipper-shutdown-problem-my-solution/7623). Untested by me. | Free — re-route cables |

On the heat half: while measuring [Fix 9](#fix-9--your-exhaust-fan-is-pid-controlled-by-a-cpu-it-isnt-pointed-at) I ran a real print with the bottom-plate fan removed — which is **how a stock SV08 leaves the factory**, because the PA1 header ships with nothing plugged into it. The host CPU reached **71.2 °C** and the mainboard MCU **64.1 °C and still climbing**. The forum thread that bolted a 160 CFM fan under the machine reports host **60 → 40 °C** and MCU **50 → 29 °C** — the same size of effect, in the same direction. That is not a temperature you can wave away.

So the honest ordering is: **check the eMMC first, because it's free** ([sysfs](#check-yours-before-you-buy-anything), then Fix 7). If the module is healthy and Fix 7 didn't help, **then** cool the board — and know that a stock machine has no board cooling at all.

What I still believe, and what I'd retract:

* **Retract:** "a fan never fixes it, reseating the eMMC is the real change." Sometimes the fan *is* the fix. I had a bad module and generalised from it.
* **Stand by:** replacing the eMMC without doing Fix 7 first is spending money to buy yourself the same problem again, more slowly.

### The part that makes this Sovol's fault

See [Fix 7](#fix-7--power-loss-recovery-writes-to-flash-on-every-single-move) — Sovol's power-loss-recovery patch writes a file to that eMMC **on every single G1 move**. A print is millions of moves. That is millions of small synchronous writes to a cheap eMMC that has no wear-leveling headroom to spare.

That workload — a forced, blocking write per move — is the worst case for a small eMMC. It's what turns a merely *slow* module into a print-killer, and over time it's a plausible way to wear one out. **Fix 7 is free. Do it before you spend money on flash.** If you have already replaced your eMMC, apply Fix 7 anyway, or the new module gets the same treatment.

### Check yours before you buy anything

The eMMC will tell you it's dying — nobody thinks to ask it. You don't need `mmc-utils`; the kernel exposes it in sysfs:

```bash
ssh sovol@<printer-ip>
cat /sys/block/mmcblk2/device/{name,date,life_time,pre_eol_info}
```

On a healthy printer (this is my current, replacement module):

```
A3A551            # part name (manfid 0xd6 = Foresee)
01/2022           # manufacture date
0x01 0x00         # life_time: Type A / Type B
0x01              # pre_eol_info
```

How to read it:

| Value | Meaning |
|---|---|
| `pre_eol_info: 0x01` | **Normal** — reserve blocks healthy |
| `pre_eol_info: 0x02` | **Warning** — 80% of reserve blocks consumed |
| `pre_eol_info: 0x03` | **Urgent** — replace it now |
| `life_time: 0x01` | 0–10% of rated write life used |
| `life_time: 0x02` | 10–20% used … and so on, in 10% bands |
| `life_time: 0x0b` | Exceeded its rated lifetime |

`pre_eol_info` is the one that matters. **`0x02` or `0x03` and you have found your problem.**

**But a clean reading does not clear the module.** Wear counters say nothing about *speed*, and a healthy-but-slow 8 GB eMMC will still stall the host under Fix 7's write-per-move. So: **apply Fix 7 first** (free, five minutes). If the shutdowns survive that, the eMMC is the next thing to change, wear counters or not.

Also look for the flash complaining out loud:

```bash
dmesg | grep -iE 'mmc[0-9].*(error|timeout|retry|crc|fail)'
```

Anything here — timeouts, CRC errors, retries — is your smoking gun. A healthy machine prints nothing.

### Fix

Replace the eMMC module. It's a standard socketed module on the CB1-style SBC. The stock part is **8 GB**; I fitted a **32 GB** one from eBay, and Sovol sell a 32 GB module themselves. Bigger is not the point — *faster* is, and on eMMC the two tend to travel together. Then reflash the Sovol image (or go [mainline](https://github.com/Rappetor/Sovol-SV08-Mainline)) and restore your config backup.

Before you refit the cover: **apply [Fix 7](#fix-7--power-loss-recovery-writes-to-flash-on-every-single-move)**, so the new module doesn't get chewed up the way the old one did.

### The bigger module is worth it anyway — and it's not just space

The stock 8 GB is cramped the moment you actually use the printer. Mine, on the 32 GB module:

```bash
df -h /                                                    # 29G total, 11G used, 18G free
du -sh ~/printer_data/gcodes ~/printer_data/timelapse      # 5.1G gcodes, 1.6G timelapse
```

**6.7 GB of gcode and timelapses** — on an 8 GB module that is not merely tight, it is impossible once the OS has taken its cut. You end up deleting prints to make room for prints, and timelapse ([Fix 1](#fix-1--timelapse-auto-render-resets-to-off-after-every-reboot)) is the first thing you give up.

And there's a nastier feedback loop: **a nearly-full eMMC is a slower eMMC.** Less free space means less spare area for the controller to work with, more write amplification, and worse write latency — the exact quantity that decides whether [Fix 7](#fix-7--power-loss-recovery-writes-to-flash-on-every-single-move)'s forced write-per-move stalls the host hard enough to shut the MCU down. A full 8 GB module is the worst possible case for this bug: small, slow, *and* out of room.

So a bigger module buys you three things at once — space for your files, headroom that keeps the flash fast, and more flash to spread the wear across.

### Verify

```bash
cat /sys/block/mmcblk2/device/pre_eol_info     # want 0x01
cat /sys/block/mmcblk2/device/life_time        # want 0x01 0x00
dmesg | grep -ic mmc                           # want no errors
```

Then run the longest print that used to kill it. The real verification is a job that completes.

### Revert

Keep the old module. If the new one changes nothing, you've lost the price of an eMMC and eliminated a variable — swap it back and go looking at the toolhead USB cable, which is the next most likely stall.

---

## Fix 7 — Power-loss recovery writes to flash on every single move

### Symptoms

* Prints with **organic supports**, **spiral z-hop**, or lots of small round moves die partway through:

  > **MCU 'extra_mcu' shutdown: move queue overflow** — or **Timer too close**

* Simple, boxy prints are fine. The more small XYZ moves your model has, the likelier it dies.
* Over months: the eMMC wears out and the printer starts shutting down at random ([Fix 6](#fix-6--timer-too-close-is-the-emmc-not-a-hot-mainboard)).

This is [issue #33](https://github.com/Sovol3d/SV08/issues/33) on Sovol's own tracker. It is open, has zero comments, and has never been answered.

### Cause

Sovol patched Klipper's `cmd_G1` to support power-loss recovery. Here's what their patch does on **every G1 move that carries X, Y and Z together** — which, with z-hop enabled, is *every move you print*:

```python
def cmd_G1(self, gcmd):
    params = gcmd.get_command_parameters()
    if 'G' in params and 'X' in params and 'Y' in params and 'Z' in params:
        if self.v_sd.cmd_from_sd:
            cfg_file_path = '/home/sovol/printer_data/config/saved_variables.cfg'
            _config = configparser.ConfigParser()
            _config.read(cfg_file_path)                 # disk read + full INI parse
            ...
            with open("/home/sovol/sovol_plr_height", 'w') as height:
                json.dump(content, height)
                height.flush()
                os.fsync(height.fileno())               # force it down to physical flash
```

Read that again: a **config-file parse**, a **JSON write**, and an **`os.fsync()`** — per move.

`os.fsync()` blocks until the data is physically on the flash. It deliberately defeats every OS write cache. So Klipper's host process — the one whose *entire job* is feeding moves to the MCU ahead of time — stops dead and waits on the eMMC, thousands of times per minute. When it can't feed the MCU fast enough, the MCU runs dry and shuts down: **`move queue overflow`**, or **`Timer too close`**.

Two consequences, both bad:

1. **It kills prints now.** Exactly the prints Sovol's own issue #33 names: organic supports, spiral z-hop.
2. **It kills your eMMC slowly.** Millions of forced flash writes per print, on a cheap module with no wear-leveling headroom. See [Fix 6](#fix-6--timer-too-close-is-the-emmc-not-a-hot-mainboard) — this is very likely what wears them out.

The bitter part: **the recovery it's paying all this for doesn't work.** The write *is* the failure mode.

### The SV08 Max ships the same patch — with a *looser* guard

Sovol published the Max's source too, and `cmd_G1` still calls `os.fsync()`. What changed is the condition in front of it:

```python
# Sovol3d/SV08 ....... klippy/extras/gcode_move.py
if 'G' in params and 'Z' in params and 'X' in params and 'Y' in params and 'F' in params:

# Sovol3d/SV08MAX .... klippy/extras/gcode_move.py     <- 'F' guard GONE
if 'G' in params and 'X' in params and 'Y' in params and 'Z' in params:
```

They **removed** a condition. Fewer moves are filtered out, so a strictly larger set of G1s now triggers the blocking write. (The firmware actually shipping on my SV08 is already the second form — which means the public `Sovol3d/SV08` repo is *stale*, and the guard on real hardware is the loose one.)

This is a two-year-old bug, carried forward into a new flagship, on a bigger machine that moves further per layer. Apply this fix on a Max too.

### Fix

Comment the block out. Back up first — this is Klipper source, not a config file.

```bash
ssh sovol@<printer-ip>
cp ~/klipper/klippy/extras/gcode_move.py ~/gcode_move.py.bak.$(date +%Y%m%d-%H%M%S)
nano ~/klipper/klippy/extras/gcode_move.py
```

In `cmd_G1`, comment out the whole block — from `if 'G' in params and 'X' in params ...` down to and including `os.fsync(height.fileno())`. On my fork (`v0.12.0-0-g02eeceb-dirty`) that's **lines 135–158**; check yours, don't blind-patch by line number.

```python
    def cmd_G1(self, gcmd):
        # Move
        params = gcmd.get_command_parameters()
        # --- SOVOL PLR DISABLED (Sovol3d/SV08 issue #33) ---
        # This block ran configparser + a json write + os.fsync() on EVERY G1
        # carrying X, Y and Z. The blocking fsync starves the MCU and hammers
        # the eMMC. PLR never worked anyway: the write WAS the failure.
        # if 'G' in params and 'X' in params and 'Y' in params and 'Z' in params:
        #     ...
        #         with open("/home/sovol/sovol_plr_height", 'w') as height:
        #             json.dump(content, height)
        #             height.flush()
        #             os.fsync(height.fileno())
        try:
            for pos, axis in enumerate('XYZ'):
                ...
```

Check it still parses, then restart:

```bash
python3 -m py_compile ~/klipper/klippy/extras/gcode_move.py && echo OK
sudo systemctl restart klipper
```

**You are giving up power-loss recovery.** That's the trade, and it's a good one: it never worked, and it's what's killing your prints and your flash. If you want real PLR, go [mainline](https://github.com/Rappetor/Sovol-SV08-Mainline).

### Verify

Klipper comes back healthy:

```bash
curl -s http://<printer-ip>:7125/printer/info | python3 -m json.tool   # "state": "ready"
```

Then prove the write is actually dead. Note the file's fingerprint, run **a real print** (the write only fires for gcode coming from the virtual SD — console moves never triggered it), and check it again:

```bash
stat -c %y /home/sovol/sovol_plr_height && md5sum /home/sovol/sovol_plr_height
# ... run a print with z-hop / XYZ moves ...
stat -c %y /home/sovol/sovol_plr_height && md5sum /home/sovol/sovol_plr_height
```

**Identical mtime and md5 across a print = fixed.** The file will keep its last pre-patch value forever, which is exactly right. Before the patch, that timestamp moved constantly during a print.

### Revert

```bash
cp ~/gcode_move.py.bak.<timestamp> ~/klipper/klippy/extras/gcode_move.py
sudo systemctl restart klipper
```

Note that a Klipper update from Sovol will also restore their version — and re-break it.

---

## Fix 8 — `M106 P1` does not do what you think, and nobody documents the fans

### Symptoms

You want to run the two part-cooling fans at different speeds — one blows across the part, the other at it — so you send `M106 P1 S128`. **Both** fans go to half speed. `M106 P0 S255` does the same thing in reverse. There is no combination of `P` that addresses one fan.

Meanwhile a fan you never asked for is audibly running while Mainsail shows every fan at `0%`, and two more sit permanently at 10% and never stop. Nobody can tell you which fan is which, because Sovol ships no fan documentation at all.

### The map (there are five fans, not two)

Read straight off a stock machine. Only the first two are yours to control:

| Name | Klipper type | Pin | What it physically is | Tach | Reachable by `M106`? |
|---|---|---|---|---|---|
| `fan0` | `fan_generic` | `extra_mcu:PA7` | part cooling, **rear** — blows *across* the part | no | only together with `fan1` |
| `fan1` | `fan_generic` | `extra_mcu:PB1` | part cooling, **front** | no | only together with `fan0` |
| `hotend_fan` | `heater_fan` | `extra_mcu:PA6` | heatbreak fan — thermostatic, kicks in at extruder ≥ 45 °C | **yes** | **no** |
| `fan2` | `temperature_fan` | `PA1` (main MCU) | **the mainboard fan** — the loud 24 V 4010 under the board, and the only fan cooling the mainboard. PID to 50 °C on `temperature_mcu`. See **[Fix 9](#fix-9--your-exhaust-fan-is-pid-controlled-by-a-cpu-it-isnt-pointed-at)** | no | **no** |
| `fan3` | `temperature_fan` | `PA2` (main MCU) | **the exhaust fan header** — but Sovol PIDs it to 60 °C on `temperature_host`. There is no host fan on this machine. See **[Fix 9](#fix-9--your-exhaust-fan-is-pid-controlled-by-a-cpu-it-isnt-pointed-at)** | no | **no** |

Four things fall out of that table immediately:

- **The fan you hear while everything reads 0% is `hotend_fan`.** It's a `heater_fan` tied to the extruder — it turns itself on at 45 °C and is invisible to `M106` and to the fan sliders. Working as designed. Stop looking for the bug.
- **`fan2` and `fan3` never stop** because both have `min_speed: 0.1`. A permanent 10% floor. That is not a stuck fan.
- **`hotend_fan` is the only fan on this machine with a tachometer** (`extra_mcu:PA1`). `fan0`/`fan1` report `"rpm": null` forever — you cannot detect a dead part-cooling blower in software. Check it by eye.
- **The Klipper names do not line up with the mainboard silkscreen.** `fan0` and `fan1` are on the **toolhead** board — the `extra_mcu:` prefix is the tell. The mainboard has its own headers silkscreened `FAN1`/`FAN2`/`FAN3`, and Klipper only ever drives two fans down there: `PA1` and `PA2`, which it calls **`fan2` and `fan3`**. So if you drive `fan1` and stare at the mainboard waiting for something to move, nothing will — you're watching the wrong board.

Here is the whole mainboard, decoded — every row verified on hardware by driving the pin and watching what responded. **Three headers marked `FAN*`, but only two fans, and neither carries the number printed next to it:**

| Silkscreen | MCU pin | Klipper object | Fitted from the factory? |
|---|---|---|---|
| `FAN1` | `PA1` | `temperature_fan fan2` | Yes — **the mainboard fan.** A cheap 24 V 40×10, and [the loudest thing on the printer](#the-other-mainboard-fan--the-one-that-screams). It is the only fan cooling the mainboard. |
| `FAN2` | `PA2` | `temperature_fan fan3` | Yes — **the exhaust fan**. |
| `FAN3` | `PA3` | `output_pin main_led` | Occupied — but **not by a fan.** It drives **the LED**. The wire in it is labelled `LED1`. |

**Only two of the three headers silkscreened `FAN*` are fans.** `FAN3` is a fan label printed above the LED connector. If you are chasing a "third fan" on this board, stop: it does not exist.

That last row is confirmed on hardware, not inferred — pulse the pin and the light blinks with it:

```bash
# LED off / on. Stock value is 1.0 (on) -- put it back when you're done.
curl -s -X POST "http://<printer-ip>:7125/printer/gcode/script?script=SET_PIN%20PIN%3Dmain_led%20VALUE%3D0"
curl -s -X POST "http://<printer-ip>:7125/printer/gcode/script?script=SET_PIN%20PIN%3Dmain_led%20VALUE%3D1"
```

Sovol's config comments (`#header 1 from the left`, `#header 2 from the left`) count the same way the silkscreen does. The *names* are what's shifted: **board `FAN1` is Klipper `fan2`, board `FAN2` is Klipper `fan3`.** Check all of this before you unplug anything in anger.

### Cause — the `M106` macro

The SV08 has no `[fan]` section, so stock `M106` has nothing to drive. Sovol bridges that with a macro (`Macro.cfg`, ~line 653):

```jinja
[gcode_macro M106]
gcode:
    {% set fan = 'fan' + (params.P|int if params.P is defined else 0)|string %}
    {% set speed = (params.S|float / 255 if params.S is defined else 1.0) %}
    {% if fan == 'fan3'%}
            SET_FAN_SPEED FAN={fan} SPEED={speed}
    {% else %}
        SET_FAN_SPEED FAN={'fan0'} SPEED={speed}
        SET_FAN_SPEED FAN={'fan1'} SPEED={speed}
    {% endif %}
```

It parses `P` into a fan name, then **throws it away**. Only `fan3` is special-cased; every other value of `P` — including `P0` and `P1`, the two fans you actually have — falls into the `else` and sets *both*. The `P` parameter is decorative.

And the one branch that does use `P` is **dead**: `SET_FAN_SPEED` only accepts a `fan_generic`, but `fan3` is a `temperature_fan`, so `M106 P3` errors out with an unknown-fan complaint. That branch was written for `[fan_generic fan3] # exhaust fan` — which is sitting **commented out** in `printer.cfg`, on pin `PA2`. The same `PA2` that `temperature_fan fan3` now owns.

> **Read that last part before you wire up an exhaust or filter fan.** `PA2` is not a free header — `[temperature_fan fan3]` already owns it. Uncomment `[fan_generic fan3]` on top of it and you have two config sections fighting over one pin. You have to *delete* the `temperature_fan`, not add alongside it. [Fix 9](#fix-9--your-exhaust-fan-is-pid-controlled-by-a-cpu-it-isnt-pointed-at) does exactly that.

### Fix

There isn't one to apply — and that's the point. Nothing here is broken hardware; it's undocumented behaviour that sends people chasing ghosts. What you do instead:

**To drive one part fan, bypass `M106` entirely** and talk to the fan directly:

```gcode
SET_FAN_SPEED FAN=fan0 SPEED=0.5     ; rear fan only, 0.0–1.0 (not 0–255)
SET_FAN_SPEED FAN=fan1 SPEED=1.0     ; front fan only
```

Put those in your slicer's custom gcode if you want asymmetric cooling. `M106` will always hit both.

**`hotend_fan` is closed-loop and it is right — leave it alone.** If you need to move a `temperature_fan` target it's `SET_TEMPERATURE_FAN_TARGET`, not `SET_FAN_SPEED`.

> **`fan2` and `fan3` are both wrong, and [Fix 9](#fix-9--your-exhaust-fan-is-pid-controlled-by-a-cpu-it-isnt-pointed-at) is the correction.** I originally wrote here that the stock 50 °C / 60 °C targets were sane. They are not. **`fan3`'s 60 °C target sits *below* where its sensor idles**, so the exhaust pegs at 100% and never stops. **`fan2`'s 50 °C target sits *at* where its sensor idles**, so the board fan loafs at 10% and never starts. One fan can never reach its target and one has never left it — and neither number is a tuning problem.

> **Beware `TARGET=0` on a `temperature_fan`.** It does not mean "full speed because the target is unreachable". It **disables the fan** — duty `0.00`. `TARGET=1` is what gives you 100%. This is the opposite of most people's first guess, and it is a good way to silently cook your mainboard while you experiment.

### Verify

No SSH needed — ask the printer what its fans are doing:

```bash
curl -s "http://<printer-ip>:7125/printer/objects/query?fan_generic%20fan0&fan_generic%20fan1&heater_fan%20hotend_fan&temperature_fan%20fan2&temperature_fan%20fan3&extruder" | python3 -m json.tool
```

On an idle, cold machine you should see `fan0`/`fan1` at `0.0` with `"rpm": null`, `hotend_fan` at `0.0`, and `fan2`/`fan3` both at exactly `0.1` with their live temperatures well under target. Heat the hotend past 45 °C and `hotend_fan` jumps to `1.0` with a real rpm — while both part fans stay at `0.0`. That's the whole confusion, visible in one command.

### Revert

Nothing to revert. This fix is knowledge.

---

## Fix 9 — Your exhaust fan is PID-controlled by a CPU it isn't pointed at

### Symptoms

A fan in the bottom of the printer runs at **full speed, all the time**. From boot. On an idle machine with cold heaters and disabled steppers. It never spins down, and it has never spun down — not once, in the life of the printer.

If you have an **enclosure**, the more expensive symptom: the chamber never gets warm. You print ABS or ASA, the enclosure is supposed to hold 40–50 °C, and it just… doesn't. Corners lift anyway. You add insulation, you seal the gaps, you check for drafts. It stays cold, because a fan is pulling the heat out of it around the clock and no one told you.

### Cause

Stock `printer.cfg`:

```ini
[temperature_fan fan3]       # "header 2 from the left"
pin: PA2
sensor_type: temperature_host
target_temp: 60
min_speed: 0.1
max_speed: 1.0
control: pid
```

Two facts that don't survive contact with each other:

1. **`PA2` is the exhaust fan header.** Sovol's own commented-out block says so, three hundred lines further down the same file — `#[fan_generic fan3] # exhaust fan` / `#pin: PA2`. The community configs agree ([vvuk/sv08-config](https://github.com/vvuk/sv08-config/blob/main/hw-fans-temps.cfg) declares `[fan_generic exhaust_fan]` on PA2), and so does Sovol's own forum ([SV08 64-bit controller pin-out](https://forum.sovol3d.com/t/sv08-64bit-controller-pin-out/5333)). **There is no fan dedicated to the host CPU on this machine**, and the exhaust is in a different compartment from it. The only thing that moves air over that chip is the **mainboard fan on `PA1`** — a fan bound to a *different* sensor entirely.

2. **`temperature_host` idles at 63–67 °C**, and the target is **60 °C**.

So the target sits *below where the sensor lives*. The PID error is positive forever, and the controller does the only thing it can: **peg the fan at 100% and hold it there permanently.** It is trying to cool the CPU by blowing air out the back of the printer. It will try until the fan dies.

This is a closed loop with **no authority**: the actuator cannot move the sensor, by any physical mechanism. Not a tuning problem. There is no value of `pid_Kp` that fixes a fan pointed at the wrong thing.

### The *other* mainboard fan — the one that screams

`PA1` is the **mainboard fan** header, and Sovol does fit a fan to it: a cheap **24 V 40×10**, sitting under the board. It is the loudest thing on the machine — by common account roughly **80% of the printer's noise**, and it starts the moment you power on.

Whether it is under control depends on which config you have, and this is where Sovol's own versions disagree with each other:

| | Sovol's published config ([Sovol3d/SV08](https://github.com/Sovol3d/SV08/blob/main/home/sovol/printer_data/config/printer.cfg)) | Shipped 2.3.3 image |
|---|---|---|
| `PA1` — mainboard fan | **no block at all** — the fan is simply not controlled | `[temperature_fan fan2]`, PID → 50 °C |
| `PA2` — exhaust fan | `[fan_generic fan3]` — a plain fan you command | `[temperature_fan fan3]`, PID → `temperature_host` |
| `[temperature_sensor mcu_temp]` | present | **commented out** |

Read that carefully, because it is the opposite of what you'd expect: **the newer image fixed the noise and broke the exhaust.** Adding the `PA1` PID is what stops that 4010 from howling. Rebinding `PA2` to the host CPU is what pinned the exhaust at 100% forever. One step forward, one step back, in the same release.

> **Do not "fix" this by reverting to Sovol's published config.** It has nothing on `PA1` — you would hand back the PID that tames the mainboard fan and get the screaming back. Keep `[temperature_fan fan2]` exactly as it is. **`PA2` is the only block that needs changing.**

**The noise fix, while you're in here:** the stock 4010 is the part everyone replaces. The drop-in is a **Noctua NF-A4x10 — the 24 V version** (`PA1` is a 24 V rail; a 12 V Noctua there is over-volted). It ships with a Molex→JST-XH adapter that plugs straight into the header. Mounts for bigger, slower fans are also popular ([BrewNinja 4010](https://www.printables.com/model/908513-sovol-sv08-mainboard-4010-fans), [EsZett 120 mm](https://www.printables.com/model/927060-sovol-sv08-mainboard-120mm-fan-mod), [kampfwuffi low-profile](https://www.printables.com/model/1048947-low-profile-mainboard-fan-mount-for-sovol-sv08)).

Whatever you fit, `PA1`'s existing PID drives it with **no config change** — and the orientation matters more than the model.

#### Fit it blowing *in*, not out

None of those mod listings say which way round the fan goes, and it is easy to get wrong — because **the wrong way still works.** That is what makes it worth a paragraph.

**This fan is the only thing that moves air through the electronics bay.** The big rear exhaust does not help you here — it vents the *print chamber*, a different compartment. Nothing else in the bay moves air at all.

So mount it blowing **outward** and it becomes an extractor with no intake. The bay drops to negative pressure and pulls its replacement air in through whatever gaps, seams and cable pass-throughs leak best. Air does cross the board — but along whatever path happens to leak, not over the hot parts. It cools. It cools *badly*, and it gives you no hint that anything is wrong.

Mount it blowing **inward, onto the board**, and the air lands where the heat is: straight onto the CPU and the stepper drivers, then out through the same gaps. Same fan, same power, air actually where it's needed.

The fan frame carries two moulded arrows — one for rotation, one for airflow. **Point the airflow arrow at the mainboard.**

> **If yours is in backwards, unscrew it and physically rotate it 180°.** Do *not* try to fix it by swapping the two wires: reversing polarity on a brushless DC fan does not reverse its direction, it simply stops it from starting, and leaving a stalled coil energised can destroy the fan's driver. There is no wiring trick. Flip the hardware.

**Measured on this machine, both ways round.** Backwards, the fan still pulled the MCU down about **5 K** — that 5 K is what the leak paths were worth. Turned around to blow in, the *same fan at the same speed* is worth **8.7 K on the MCU and 11.7 K on the host CPU** (full numbers below). **Flipping it roughly doubled it.** It is the third thing on this board that works just well enough to hide the fact that it's wrong.

### The measurement

Steppers disabled, heaters off, idle. **The exhaust (`fan3`, PA2) was pinned at duty 1.00 for the entire run** so its PID could not absorb the change into its own duty instead of into temperature. Only the bottom-plate fan (`fan2`, PA1) was varied. Each phase ran 8 minutes; the figure is the mean of the last 3.

> **Soak first, or you will measure nothing but the board warming up.** After a power cycle the MCU climbs for a quarter of an hour. Start recording immediately and that warm-up lands in your baseline and flatters the next phase — it is how an earlier version of this table produced a number I had to withdraw. This run held phase-A conditions for **15 minutes before recording anything**, until drift fell from +1.59 K/min to +0.12 K/min.

> **Reproducible on a stock SV08 — I said otherwise here, and I was wrong.** An earlier version of this section claimed `PA1` ships empty and that the fan I measured was an aftermarket addition. It isn't. **`PA1` ships with Sovol's own loud 24 V 4010**, and the Noctua on my machine is a *replacement* in the same header, on the same PID. So you can run this measurement on a stock printer today — you will just do it with a noisier fan. What the mod changes is the noise, not the wiring.

| Phase | bottom fan (`fan2`, PA1) | exhaust (`fan3`, PA2) | MCU | host CPU |
|---|---|---|---|---|
| **A** | idle (0.10 floor) | pinned 1.00 | 47.74 °C | 62.67 °C |
| **B** | **1.00** (blowing in) | pinned 1.00 | **39.07 °C** | **50.96 °C** |
| | | | **−8.67 K** | **−11.72 K** |

**The bottom-plate fan is worth 11.7 K on the host CPU.** The exhaust never moved. A fan blowing over the *electronics bay* is the only thing cooling that chip — obvious in hindsight, since it is the only thing moving air across it at all.

And these are **idle** figures. A fan's effect grows with the temperature difference it works across, and the host's real problem is mid-print — bed at 60–100 °C radiating into the bay, steppers and drivers dumping heat in, host past 70 °C. **Treat 11.7 K as a floor, not a ceiling.**

#### The trap this springs

Watch what the machine did the instant the test released it back to stock:

```
fan3 (exhaust):  target 60.0   duty 0.10   host 51.5 °C
```

The chamber exhaust dropped to **10%**. Not 100% — *10%*.

Because the bottom fan had pulled the host CPU down to 51 °C, and the exhaust's PID target is 60 °C. Setpoint reached, so the loop backed off. The exhaust had only ever run flat out **because the host was too hot to ever reach target.**

> **Cool your electronics properly and the chamber exhaust switches itself off.**

Chamber venting on this printer is slaved to how well an unrelated fan in another compartment cools the CPU. The better your board cooling gets, the less your chamber gets vented — and nothing in the UI will ever tell you that is why. This is the single strongest argument for the config change below: as long as `PA2` is a `temperature_fan` bound to `temperature_host`, **your chamber exhaust is not yours to control.**

**Independent proof, from an earlier run:** starve the exhaust to its 0.10 floor and the host *keeps getting colder* (−2.02 K, still falling). Remove the fan supposedly responsible for a temperature, and that temperature is unaffected. It was never cooling anything in there.

And the moment that makes it undeniable — in phase B, as the *board* fan dragged the host down through 60 °C, watch the exhaust fan's own PID finally back off:

```
t=423s   host 60.97 °C    fan3 duty 0.98
t=443s   host 60.00 °C    fan3 duty 0.78
t=463s   host 59.84 °C    fan3 duty 0.45
t=483s   host 59.35 °C    fan3 duty 0.10   <-- target reached, for the first time ever
```

**The exhaust fan hit its setpoint exactly once in its life — and only because a different fan did the work for it.**

> **What this means if your machine is stock.** You have the same fan on `PA1` that I do — Sovol's, not a Noctua — driven by the same PID, whose `target_temp: 50` sits *at* the MCU's idle temperature. So it loafs at its 10% floor and never spins up, the host never falls below 60 °C, the exhaust PID never sees a negative error, and **duty never leaves 1.00.** The throttle-down above only happened because I forced that fan to full. Stock, you get the pinned exhaust and none of the cooling — the hardware to fix it is already bolted to your printer, idling.

### Fix

Both mainboard fans are wrong, in opposite directions, and you have to fix them together — because **fixing one on its own breaks the other.** That is the whole trap, and it is why this section is longer than you want it to be.

* `PA1` (the board fan) is PID'd to a target it has already reached, so it **never spins up.**
* `PA2` (the exhaust) is PID'd to a target it can never reach, so it **never spins down.**
* Pin `PA1` to full — the obviously correct thing — and the host CPU drops below 60 °C, which is the *exhaust's* setpoint. The exhaust promptly throttles itself to 10% and **stops venting your chamber.**

So: take both fans out of the PID's hands.

#### 1. The board fan — pin it at 100% and hide it

Replace `[temperature_fan fan2]` with a plain always-on pin. There is no temperature at which you want *less* airflow over your own mainboard, so there is nothing to control and nothing to put in the UI:

```ini
[output_pin _mainboard_fan]
# PA1 = board silkscreen FAN1 = the fan under the mainboard.
# Cools the mainboard MCU *and* the host CPU. Nothing else in the bay moves air.
# Held at 100%: there is no reason to ever run it slower.
# Leading underscore hides it from Mainsail - there is nothing to adjust.
# shutdown_value 1.0 = keeps cooling the board even if Klipper shuts down.
pin: PA1
value: 1.0
shutdown_value: 1.0
```

**Do this only if your fan is quiet.** Pinning Sovol's stock 4010 at 100% forever means it screams at you forever. Fit the Noctua first (above), *then* pin it. If you are staying on the stock fan, leave `[temperature_fan fan2]` alone and drop its `target_temp` to `45` instead — it will at least wake up.

**A `shutdown_value` of `1.0` is deliberate.** Klipper's emergency shutdown is exactly when you want the board still being cooled.

#### 2. Give the MCU its temperature back

The `temperature_fan` you just deleted was the only thing putting the mainboard MCU on screen. Sovol ships the proper sensor for it **commented out**. Uncomment it:

```ini
[temperature_sensor MCU_Temp]
sensor_type: temperature_mcu
min_temp: 0
max_temp: 100
```

Now the chip reads out next to the toolhead's, as a temperature — which is all it ever was.

#### 3. The exhaust — make it a fan you command

```ini
[fan_generic Chamber_Exhaust]
# PA2 = board silkscreen FAN2 = big rear fan. Vents the PRINT CHAMBER, nothing else.
# Was a [temperature_fan] PID'd against the HOST CPU - a chip it is not near and
# cannot cool. That made chamber venting a side effect of CPU temperature.
pin: PA2
kick_start_time: 0.5
off_below: 0.05
```

This also un-breaks the dead `M106 P3` branch from [Fix 8](#fix-8--m106-p1-does-not-do-what-you-think-and-nobody-documents-the-fans) — `SET_FAN_SPEED` refuses a `temperature_fan`, but a `fan_generic` is exactly what it wants. Point the branch at the new name in `Macro.cfg` (do this in **both** `M106` and `M107`; the `fan == 'fan3'` test stays — it is the *P-index* test, not the fan's name):

```jinja
    {% if fan == 'fan3'%}
            SET_FAN_SPEED FAN=Chamber_Exhaust SPEED={speed}
    {% else %}
```

`M106 P3 S255` starts working the day you make this change. So does your slicer.

#### 4. A control a human can use

A rear fan at 100% is loud. Give yourself 10% steps, with a real OFF for ABS:

```ini
[gcode_macro EXHAUST]
description: Chamber exhaust speed, 10% steps. S=0 is a real OFF (use for ABS).
gcode:
    {% set s = params.S|default(100)|int %}
    {% if s <= 0 %}
        {% set s = 0 %}
    {% else %}
        {% set s = [[(((s + 5) // 10) * 10), 10]|max, 100]|min %}
    {% endif %}
    SET_FAN_SPEED FAN=Chamber_Exhaust SPEED={ s / 100 }
    RESPOND MSG="Chamber exhaust: {s}%"
```

`EXHAUST S=70`. Anything above zero clamps to a 10% floor; `S=0` is the deliberate exception, so ABS can have a genuinely still chamber.

#### 5. The sticky-fan trap — the part everyone will skip

**Read this one.** A `fan_generic` has **two** default behaviours, and neither is the one you want. Both come straight from Klipper's [`fan.py`](https://github.com/Klipper3d/klipper/blob/master/klippy/extras/fan.py):

* **At boot it is OFF.** `last_fan_value` initialises to `0.`, and `_handle_request_restart()` calls `set_speed(0.)` — so a reboot or a `FIRMWARE_RESTART` leaves the exhaust at **zero**.
* **Between prints it is sticky.** Nothing in Klipper resets a `fan_generic` when a job ends. Whatever the last job set, the next job inherits.

Now look at what the slicer actually emits:

* **ABS/ASA** with air filtration configured: `M106 P3 S0` at the start. Exhaust → **0**.
* **PLA/PETG**: **nothing at all.** I grepped a real PETG job off the printer — zero `M106 P3` anywhere in the file.

Put those together. Print one ABS part, then print PLA. The ABS job set the exhaust to 0; the PLA job never mentions it; **the fan stays at 0 and you never find out.** Your chamber silently stops venting, job after job, until you happen to look.

So you need **both** halves: a default asserted at boot (because boot leaves it off), and a hand-back after every job (because jobs are sticky). Boot:

```ini
[delayed_gcode _exhaust_boot_default]
# A fan_generic comes up at 0 after boot/FIRMWARE_RESTART. Assert the default.
initial_duration: 3.0
gcode:
    SET_FAN_SPEED FAN=Chamber_Exhaust SPEED=1.0
```

And the same line at the end of **`END_PRINT`** *and* **`CANCEL_PRINT`** in `Macro.cfg` — a cancelled ABS print leaves the fan at 0 exactly like a finished one:

```ini
    SET_FAN_SPEED FAN=Chamber_Exhaust SPEED=1.0
```

> **Why the end of the print and not the start.** Putting the reset in your start macro looks equivalent and isn't. It bets on the slicer emitting its air-filtration command *after* the start macro runs — and if that bet loses, your ABS print gets a 100% exhaust and warps. Restoring at the *end* cannot lose that race: whatever the job did to the fan, the job undoes on its way out. **Fail safe, not fail loud.**

> **Find your real start macro before you go patching one.** On the SV08 the slicer calls **`START_PRINT`** (`Macro.cfg`). There is also a **`PRINT_START`** in `plr.cfg` that nothing ever calls — patch that one and you will change nothing and believe you fixed it.

#### 6. The slicer side (OrcaSlicer)

The printer half is done; the slicer half is where you actually get ABS to work. In Orca:

1. **Printer Settings → Basic information → tick "Exhaust fan"** (`support_air_filtration`). Miss this and Orca emits **no `M106 P3` at all**, and everything below is dead config.
2. **Filament → Cooling → Fan control → Air filtration**, per filament:

| Filament | During print | On completion | Why |
|---|---|---|---|
| **ABS / ASA** | **0%** | **100%** | A vented chamber is a cold chamber, and a cold chamber warps ABS. Purge the fumes *after*. |
| **PLA / PETG** | 100% | 100% | Nothing to hold in. Vent it. |

> **Orca ships ABS, ASA, PC and PAHT-CF profiles with air filtration ON at 70%** — during the print *and* after it ([OrcaSlicer #9002](https://github.com/SoftFever/OrcaSlicer/issues/9002), open, unassigned). Those are exactly the filaments that need a *hot* chamber. It is backwards, it is the default, and it is fighting your enclosure. Change it.

### Verify

```bash
curl -s "http://<printer-ip>:7125/printer/objects/query?fan_generic%20Chamber_Exhaust&output_pin%20_mainboard_fan&temperature_sensor%20MCU_Temp" | python3 -m json.tool
```

Then walk every path — each of these should be audible:

```gcode
EXHAUST S=0        ; a real stop. Probably the first in the fan's life.
EXHAUST S=37       ; -> 0.40  (snaps to 10% steps)
EXHAUST S=3        ; -> 0.10  (floor)
M106 P3 S128       ; -> 0.50  (the slicer's path)
M107 P3            ; -> 0.00  (slicer OFF bypasses the floor)
```

Then prove both halves of the sticky trap are covered:

* **Reboot the printer** and query again: **1.00**, not `0.0`. That is `_exhaust_boot_default` doing its job.
* **Run an ABS job** (or just send `EXHAUST S=0`, then `END_PRINT`) and query: **1.00**. That is the `END_PRINT` restore, and it is what stops an ABS print from silently poisoning the next PLA one.

### The numbers

Idle, steppers off, heaters off — stock config versus the config above:

| | stock | fixed |
|---|---|---|
| Mainboard MCU | 47.7 °C | **38.1 °C** |
| Mainboard host CPU | 62.7 °C | **50.6 °C** |
| Toolhead MCU | 43.7 °C | 43.7 °C (unchanged, as expected) |

And the exhaust now sits wherever you put it, instead of wherever the CPU's temperature happens to leave it.

**Those are idle figures — a floor, not the answer.** The complaint that started all of this was a host CPU over 70 °C *during a print*, with the bed radiating into the bay. So here is the measurement that was owed: a PETG job (bed 75 °C, hotend 245 °C), enclosed, panels on, sampled every 30 s. At 32% through, with everything thermally settled, the bottom-plate fan was switched off and then back on:

| | board fan ON | board fan OFF |
|---|---|---|
| Mainboard host CPU | 59.4 °C | **71.2 °C** |
| Mainboard MCU | 47.1 °C | **64.1 °C** |

**The complaint reproduces on demand.** Take that fan away and the host goes over 70 °C under print load — 11.8 K, matching the 11.7 K seen at idle almost exactly. The mainboard MCU is the bigger surprise: **17 K**, and still climbing when the fan went back on. Recovery is fast — host back to 60.9 °C and MCU to 48.4 °C within four minutes.

**A stock SV08 ships with PA1 empty.** There is no bottom-plate fan at all. So this is not drift, and it is not a fault that developed — it is what the machine does out of the box, in an enclosure, and the fan that Sovol *did* wire up (the exhaust) is PID'd to a sensor it cannot influence. Fit a fan to PA1 and drive it from `[output_pin _mainboard_fan]` as above.

### Revert

Restore `[temperature_fan fan2]` and `[temperature_fan fan3]`, re-comment `[temperature_sensor mcu_temp]`, undo the `Macro.cfg` edits, `FIRMWARE_RESTART`. The board fan goes back to idling at 10% and the exhaust goes back to 100% forever.

### The general trap

**A `temperature_fan` whose target is below its sensor's idle temperature is not a fan — it's a fan permanently switched on, wearing out, with a PID bolted to the front of it for decoration.** Klipper will not warn you. Mainsail will cheerfully render it as 100% and you'll assume that's the machine working hard.

Before you trust *any* closed loop, ask the boring question: **can this actuator actually move this sensor?** If the answer is no, the loop is a lie, and no amount of tuning will make it true.

---

## Fix 10 — One bounce of the filament switch pauses your print

### Symptoms

A print pauses on its own, mid-job. The toolhead parks, Z lifts, and the job sits there cooling. There is filament in the sensor. There was filament in the sensor the whole time. Mainsail shows no error — the printer is `ready`, `print_stats.message` is empty, and the console says only:

```
// Pause Print!
```

Resume, and it prints on happily. It may not happen again for days.

### Cause

Stock config:

```ini
[filament_switch_sensor filament_sensor]
pause_on_runout: True
event_delay: 3.0
pause_delay: 0.5
switch_pin: PE9
```

`pause_on_runout: True` hands the decision to Klipper's built-in handler, and that handler **pauses on a single read of the switch.** PE9 is a mechanical lever. A machine that slams a 300 mm gantry around at 10000 mm/s² vibrates, the lever chatters, the switch reads open for a few milliseconds, and Klipper — doing exactly what it was told — pauses your print.

There is no debounce anywhere in that config. `pause_delay: 0.5` is not one: it is how long Klipper *waits before pausing*, not how long it waits before *believing* the sensor. `event_delay: 3.0` is not one either: it is the minimum gap between two events, so it stops the same bounce firing twice — it does nothing about the first one.

**The tell that it was a bounce and not a real runout:** query the sensor after the pause and it reads `filament_detected: true`. The filament never went anywhere.

```bash
curl -s "http://<printer-ip>:7125/printer/objects/query?filament_switch_sensor+filament_sensor"
```

### The mistake to avoid

The obvious fix is to re-read the switch after a short delay, and the obvious way to wait is `G4`:

```ini
# DOES NOT WORK. Two separate reasons.
[gcode_macro _RUNOUT_DEBOUNCE]
gcode:
    G4 P1200
    {% if printer["filament_switch_sensor filament_sensor"].filament_detected %}
        ...
```

**Reason one: Klipper renders the entire macro template before it executes any of it.** That `{% if %}` is evaluated at render time — *before* the `G4` has run. You get the same stale reading that caused the pause, every time. This trips people up constantly and it fails silently, which is the worst way to fail.

**Reason two: `G4` is a dwell in the motion queue.** Even if the timing worked, you would be stopping a hot nozzle over the part for over a second, on every bounce, to decide whether you needed to stop at all. You would be trading a pause for a blob.

### Fix

Turn off the built-in pause, and re-check from a **`delayed_gcode`**. A `delayed_gcode` fires from Klipper's reactor, not from the motion queue: the toolhead never stops, and the template is rendered *when it fires*, so the sensor read is fresh.

In `printer.cfg`, replace the `[filament_switch_sensor]` block with:

```ini
[filament_switch_sensor filament_sensor]
# pause_on_runout is False on purpose: the built-in pause fires on a single
# switch read, so one bounce of the PE9 lever kills the print. _runout_recheck
# re-reads the switch after settle_time and only pauses if it is still open.
pause_on_runout: False
event_delay: 3.0
pause_delay: 0.5
switch_pin: PE9
runout_gcode:
    UPDATE_DELAYED_GCODE ID=_runout_recheck DURATION={printer["gcode_macro _runout_var"].settle_time}

[gcode_macro _runout_var]
variable_settle_time: 1.2
gcode:

# Runs from the reactor, not the motion queue, so the toolhead never stops and
# no blob is left on the part. A real runout loses settle_time of air printing.
[delayed_gcode _runout_recheck]
gcode:
    {% if printer["filament_switch_sensor filament_sensor"].filament_detected %}
        {action_respond_info("Filament sensor bounced - filament still present after settle. Print continues.")}
    {% else %}
        {action_respond_info("Filament runout confirmed after settle - pausing.")}
        PAUSE
    {% endif %}
```

`FIRMWARE_RESTART`.

The empty `gcode:` under `_runout_var` is not a typo — it is the standard Klipper idiom for a macro that exists only to hold a variable. Sovol's own `_global_var` in `Macro.cfg` is written the same way.

### What this actually costs you

**On a bounce:** nothing. The toolhead never stops, you get one line in the console, the print continues.

**On a real runout:** you print `settle_time` of air — 1.2 seconds — before pausing. That is a centimetre or two of empty extrusion at the point where the filament ran out, which is precisely where you were going to have a defect anyway, and `PAUSE` parks and lifts regardless. It costs nothing that mattered.

1.2 s is a starting point, not a law. Raise `settle_time` if you still get spurious pauses; lower it if you dislike the air-printing. It is a plain variable — no macro edits.

### Verify

You cannot easily fake a bounce, so verify the two ends instead.

**A real runout still pauses.** Pull the filament out of the sensor mid-print. Expect, ~1.2 s later:

```
// Filament runout confirmed after settle - pausing.
```

**The recheck macro is live and reads the sensor correctly.** With filament loaded, fire the delayed gcode by hand — it should decline to pause:

```bash
curl -s -X POST "http://<printer-ip>:7125/printer/gcode/script?script=UPDATE_DELAYED_GCODE%20ID%3D_runout_recheck%20DURATION%3D1"
# -> // Filament sensor bounced - filament still present after settle. Print continues.
```

If that second one pauses a loaded printer, your sensor is reading open with filament in it and you have a hardware problem, not a bounce problem.

### Revert

Set `pause_on_runout: True`, delete `runout_gcode:`, `[gcode_macro _runout_var]` and `[delayed_gcode _runout_recheck]`, `FIRMWARE_RESTART`.

### The general trap

**A sensor is not a fact, and a switch is not a sensor — it is a switch, with a lever, on a moving machine.** Any handler that acts on a single sample of a mechanical contact will eventually act on a bounce. The stock config wires a one-sample read of a chattering lever directly to "destroy the current job", and the printer that shakes the lever is the same printer that is running the job.

Ask what the *cost of being wrong* is in each direction. A missed runout for 1.2 s costs two centimetres of air. A false runout costs a print. When the costs are that lopsided, sample twice.

---

## Fix 11 — Your printer announces itself to a cloud service you never signed up for

### Symptoms

None. That's the problem — this one is completely silent.

Out of the box, `moonraker-obico` is **installed, enabled and running**, and it is not linked to anything, because you have never heard of it. So it sits there doing this:

```
WARNING  obico.app - auth_token not configured. Retry after 2s
WARNING  obico.printer_discovery - HTTPSConnectionPool(host='app.obico.io', port=443):
         Max retries exceeded with url: /api/v1/octo/unlinked/
```

Every two seconds. Forever. My printer's log has that line **1,753 times**, and I never installed Obico, never opened an account, and never clicked anything.

### What it actually sends

`printer_discovery.py` POSTs to `https://app.obico.io/api/v1/octo/unlinked/`, and the payload is built by `_collect_device_info()`:

| Field | Value |
|---|---|
| `host_or_ip` | **Your printer's local IP address** |
| `hostname` | Your printer's hostname |
| `os`, `arch` | Debian / aarch64 |
| `rpi_model` | The SBC model string |
| `machine_type` | `Klipper` |
| `device_id` | Random per process start |

The *purpose* is legitimate: it's how the Obico app finds an unclaimed printer on your network and offers to claim it. That's a reasonable feature **if you asked for it**. Nobody asked for it. It is on by default, it runs unlinked, and the thing it's broadcasting is the internal address of a device on your LAN that accepts unauthenticated print commands (see [Fix 12](#fix-12--ssh-host-keys-from-the-factory-and-passwordless-root)).

Also in `moonraker-obico.cfg`, uncommented, shipped:

```ini
[misc]
sentry_opt = in
```

Crash telemetry, opted **in**, on your behalf.

### The second-order damage

Those retries are written to disk. On my machine:

```
569K  moonraker-obico.log
9.6M  moonraker-obico.log.1
9.6M  moonraker-obico.log.2
```

**Nearly 20 MB of "I can't reach a server I was never told to reach"**, written to the same cheap eMMC that [Fix 6](#fix-6--timer-too-close-is-the-emmc-not-a-hot-mainboard) and [Fix 7](#fix-7--power-loss-recovery-writes-to-flash-on-every-single-move) are about. Sovol found a *third* way to grind that flash.

### Fix

```bash
ssh sovol@<printer-ip>
sudo systemctl disable --now moonraker-obico
```

Then remove the macro include from `printer.cfg` — the Obico first-layer-scan macros get loaded into Klipper whether the service is linked or not:

```ini
# [include moonraker_obico_macros.cfg]
```

And bin the dead logs:

```bash
rm ~/printer_data/logs/moonraker-obico.log*
```

### If you actually want Obico

Then use it — but point it at **your own server**, not theirs. Obico is open source and self-hostable. Edit `moonraker-obico.cfg`:

```ini
[server]
url = http://<your-obico-host>:3334

[misc]
sentry_opt = out
```

The complaint here is not "Obico is bad". It's that a printer shipped **dialling a third-party cloud every two seconds, unlinked, with telemetry opted in, and no disclosure.**

### Verify

```bash
systemctl is-enabled moonraker-obico   # -> disabled
systemctl is-active moonraker-obico    # -> inactive
```

The log stops growing. That's it.

### Revert

```bash
sudo systemctl enable --now moonraker-obico
```

---

## Fix 12 — SSH host keys from the factory, and passwordless root

> **This is the one where I'd stop reading the fun 3D-printing article and go do something.**

### Symptoms

None, again. Everything works perfectly. That is what a security hole looks like.

### What's wrong

Three things stack, and it's the stacking that matters:

**1. The SSH host keys were generated once, at the factory, and shipped to everyone.**

```bash
$ ls -l /etc/ssh/ssh_host_*_key
-rw------- 1 root root  505 Dec 25  2023 /etc/ssh/ssh_host_ecdsa_key
-rw------- 1 root root  399 Dec 25  2023 /etc/ssh/ssh_host_ed25519_key
-rw------- 1 root root 2590 Dec 25  2023 /etc/ssh/ssh_host_rsa_key

$ ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
256 SHA256:...  root@ubuntu (ED25519)
```

**Christmas Day 2023** — the image build date, not your printer's first boot. The comment says `root@ubuntu`, which is the machine that built the image. Every SV08 in the world has **the same private host key**, and it's sitting in a public disk image. Anyone can extract it and impersonate your printer; SSH's entire man-in-the-middle protection is void, and your client will never warn you, because the key it's shown is the key it expects.

This is [Sovol3d/SV08 issue #26](https://github.com/Sovol3d/SV08/issues/26). It is **open, with zero comments**, and the one-line fix was handed to them in the issue text.

**2. The `sovol` user has passwordless sudo.**

```bash
$ sudo -n true && echo NOPASSWD
NOPASSWD
```

**3. The password is the vendor default,** and it's in the manual.

Chain them: anyone who can reach port 22 gets root on the machine that drives a 300 °C heater. And [Moonraker has no authentication at all](#the-security-holes-are-identical), so they don't even need SSH to start a print.

### Fix

**Back up the old keys first** — if something goes wrong you want to be able to put them back.

```bash
ssh sovol@<printer-ip>

sudo mkdir -p ~/ssh_hostkey_backup-$(date +%Y%m%d)
sudo cp -a /etc/ssh/ssh_host_* ~/ssh_hostkey_backup-$(date +%Y%m%d)/

sudo rm -f /etc/ssh/ssh_host_*
sudo ssh-keygen -A
sudo rm -f /etc/ssh/ssh_host_dsa_key*      # -A also mints a 1024-bit DSA key. Bin it.
sudo systemctl restart ssh
```

**This does not touch your login, your `authorized_keys`, or your password.** It changes the printer's *identity*, not your access. Klipper, Moonraker and any running print are completely unaffected — I did it mid-print.

Then change the password, because a fresh host key protecting a default password is theatre:

```bash
passwd
```

Your laptop will now — correctly — refuse to connect:

```
@@@ WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED! @@@
```

That warning is the fix working. Clear the stale entry:

```bash
ssh-keygen -R <printer-ip>
```

### Verify

```bash
$ ls -l --time-style=+%F /etc/ssh/ssh_host_ed25519_key
-rw------- 1 root root 399 2026-07-14 /etc/ssh/ssh_host_ed25519_key   # today, not Dec 2023

$ ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
256 SHA256:...  root@SV08 (ED25519)     # your machine's name, not root@ubuntu
```

### Revert

```bash
sudo cp -a ~/ssh_hostkey_backup-YYYYMMDD/* /etc/ssh/
sudo systemctl restart ssh
```

(You won't want to. This is the one fix in this document with no downside.)

### While you're here: Moonraker

`moonraker.conf` ships with `host: 0.0.0.0`, all of RFC1918 in `trusted_clients`, and no `force_logins`. **Every device on your network can move your gantry, heat your nozzle and start a print, with no credentials.** Same on the SV08 Max.

If your printer is on a network you don't fully control — a flatshare, an office, a makerspace, a guest VLAN with an IoT gadget on it — narrow `trusted_clients` to the machines that actually need it, and turn on `force_logins`. If it's on your own LAN behind your own router, this is a lower-priority cleanup than the host keys, which are broken *no matter whose network you're on*.

---

## Fix 13 — there's no one-button way to park the head for maintenance

The only entry here that isn't a defect — the one thing worth **adding** rather than fixing.
Every other job on this list undoes something Sovol got wrong; this one just fills a gap.

### Symptoms

* You want to reach the nozzle, wipe the plate, or swap a fan, and there's no button for it.
* So you hand-jog X, then Y, then Z in the Mainsail control panel, every single time.
* Do it wrong — jog Y to *max* thinking that's the front — and you drive the head into the
  back frame (see below; on the SV08 the front is Y=0, not Y_max).

### Fix — add the macro that ships in this repo

Drop in [`toolhead-maintenance.cfg`](toolhead-maintenance.cfg). It parks the toolhead at the
**front-center of the bed, at half the Z travel** — nozzle and bed both in easy reach —
with one button.

It refuses to run mid-print (`action_raise_error`), **always homes first** (`G28`) — after
a power-up Klipper has no idea where the head is, so it homes unconditionally rather than
trusting a stale position — and lifts Z *before* it moves in XY so it never drags across
the plate.

### The one thing to get right — front is Y=0, not Y_max

On the SV08 the bed's **Y=0 edge faces you**. "Move to the front" means Y at a *small*
value, not `Y_max` — `Y_max` (364 on the SV08) is the **back** of the machine. Send the
head to `Y_max` expecting the front and you'll walk it straight into the back frame. The
macro parks at `Y=5` (off the endstop, still the front). If your machine is mirrored,
change `y_front` to `(printer.toolhead.axis_maximum.y - 5)`.

Everything else is derived from the printer's own limits at runtime —
`x_mid = axis_maximum.x / 2`, `z_mid = axis_maximum.z / 2` — so the file is drop-in on an
SV08, an SV08 Max, or any other Klipper printer with no numbers to edit. On a stock SV08
(X 355, Z 347) it parks at **X 177.5, Y 5, Z 173.5**.

### Install

```bash
# 1. upload the snippet next to your other configs
curl -X POST http://<printer-ip>:7125/server/files/upload \
  -F "root=config" \
  -F "file=@toolhead-maintenance.cfg;filename=toolhead-maintenance.cfg"

# 2. add the include to printer.cfg (once)
#    [include toolhead-maintenance.cfg]

# 3. reload
curl -X POST "http://<printer-ip>:7125/printer/gcode/script?script=FIRMWARE_RESTART"
```

Or just drop the file in `~/printer_data/config/`, add `[include toolhead-maintenance.cfg]`
to `printer.cfg`, and `FIRMWARE_RESTART`. A **TOOLHEAD_MAINTENANCE** button then appears in
Mainsail's *Macros* panel.

### Verify

```bash
# button/macro registered?
curl -s "http://<printer-ip>:7125/printer/objects/list" | grep -o TOOLHEAD_MAINTENANCE

# run it, then confirm where it parked
curl -X POST "http://<printer-ip>:7125/printer/gcode/script?script=TOOLHEAD_MAINTENANCE"
curl -s "http://<printer-ip>:7125/printer/objects/query?toolhead=position,homed_axes"
```

On a stock SV08 `position` should read ~`[177.5, 5.0, 173.5, …]` and `homed_axes` `"xyz"`.

### Revert

Remove `[include toolhead-maintenance.cfg]` from `printer.cfg` and `FIRMWARE_RESTART`
(optionally delete the file). Nothing else references it.

---

## Fix 14 — `G28` homes Y into the corner that pinches the filament tube

**Cost:** free · **Severity:** damages the tube; sensorless Y false-triggers a few mm off · **Files:** `printer.cfg`

> ⚠️ **Not yet proven — test this one with a hand on the power button.**
>
> This fix is several edits deep and it changes what `G28` is allowed to do before the
> machine knows where it is. An intermediate state *can* drive the nozzle into the bed at
> full Z travel — that happened here during development, from a homing path that authorised
> a long blind descent with the head off the plate. Every homing path below has been tested
> green on one printer, but that is one printer, one firmware build
> (`v0.12.0-0-g02eeceb`), and no long-term running.
>
> Until it has more miles: stand at the machine, hand on the PSU switch or plug, for the
> first `G28`, `G28 X`, `G28 Y`, `G28 Z` and cold bare `G28` after applying it. Kill power
> the moment the toolhead moves down when it should move up, or moves in XY while off the
> plate. Do not run it unattended and do not start a print on the strength of one good home.

### Symptoms

Your filament tube is scuffed, kinked or worn where it meets the toolhead. And every so
often `G28` completes without an error but Y lands a few mm short — first layer off in Y,
a print that starts off the plate, nothing in the log.

**Both symptoms are Z-dependent.** They show up when the gantry is **high — roughly from
mid Z travel (~170 mm) upward** — and are largely absent low down. High gantry pulls the
filament tube taut against the frame in the right-rear corner; low gantry leaves it slack,
so it neither rubs nor loads the motor. So `G28` from a low gantry usually looks fine, and
`G28` at the end of a tall print — or any home that follows a Z lift — is where the tube
gets worked and Y false-triggers. Do not conclude the machine is healthy because homing
from Z 0 is clean.

### What's wrong

Stock `printer.cfg` homes **both** axes positive:

```ini
[stepper_x]
position_endstop: 355
homing_positive_dir: True

[stepper_y]
position_endstop: 364
homing_positive_dir: true
```

X to the right, Y to the rear — so every home ends in the **right-rear corner**. That is
exactly where the filament tube gets trapped between the toolhead and the frame. Every
home works the tube against the frame. That's the wear.

It also costs you accuracy, because X and Y are **sensorless** — TMC2209 StallGuard4,
`driver_sgthrs: 65`, `diag_pin: PE15`/`PE13`, `stealthchop_threshold: 0`. There is no
switch. The driver watches motor load and reports a stall as an endstop trigger. On the
TMC2209 a *higher* `SGTHRS` means *more sensitive*, and 65 does not leave much margin.
Tube drag is load. The toolhead meets that drag part-way through the approach, StallGuard
fires early, and Klipper sets Y zero **before** the toolhead ever reached the frame. It's
a false trigger, not lost steps — which is why it's silent, and why it repeats at some
gantry heights and not others. The drag scales with gantry height: above roughly mid Z the
tube is under tension and adds real load; below that it hangs slack and adds almost none.
That is the whole of the "sometimes" — it is not random, it tracks Z.

Sovol's own workaround is to lift over the problem, and it makes things worse. Their
`[homing_override]` ends with:

```ini
axes: xyz
set_position_z: 0
```

`set_position_z` tells Klipper "assume Z is 0 and stop checking it" — Klipper's docs say
so in the same paragraph: *"Setting this disables homing checks for that axis."* Sovol
applies it **unconditionally**, including when Z is homed and Klipper knows the true
height to the micron. So the `G0 Z5 F300` on the next line is not a move to Z 5. The
gantry is *actually* at, say, 160 mm, Klipper has been told 0, so "go to 5" moves **+5 mm,
to a real 165**. The limit check runs against the fabricated number and passes. Nothing is
logged.

### Fix — home into a different corner

Only X has to change. Home it to the **left**, and the head is already clear before Y ever
runs back, so the corner never happens. Y stays stock.

```ini
[stepper_x]
position_endstop: 0           # was 355
homing_positive_dir: False    # was True
```

Then rewrite `[homing_override]` — drop `set_position_z: 0`, back X off to the left, and
home **X first, then Y**:

```gcode
   {% elif not 'Z' in params and 'X' in params and 'Y' in params %}
     _PRE_HOME_LIFT
     G28 X
     G0 X7 F1200
     G4 P2000
     G28 Y
     G0 Y357 F1200
```

```ini
axes: xyz
```

**Your X origin moves.** Klipper used to be told "the trigger point is X355"; now it's
told "the trigger point is X0". Those agree only if the real travel is exactly 355 mm. It
won't be, quite. Whatever the difference is, your whole X coordinate frame shifts by it —
which puts bed mesh, `quad_gantry_level` points and probe offsets off by the same amount.
If print placement matters to you, home, jog the nozzle to the physical left edge of the
bed, read the offset and feed it back into `position_endstop`.

### Lift before every XY move, never descend blind

Homing Z on this printer *is* probing — the Z endstop is the inductive probe
(`probe:z_virtual_endstop`), so it only stops when there is metal underneath. Two rules
fall out of that, and both are load-bearing:

```gcode
[gcode_macro _PRE_HOME_LIFT]
description: Upward-only 5mm Z lift before any XY motion during homing
gcode:
   {% set th = printer.toolhead %}
   G90
   {% if 'z' in th.homed_axes %}
     {% if th.axis_maximum.z - th.position.z > 5.5 %}
       G91
       G1 Z5 F600
       G90
     {% endif %}
   {% else %}
     FORCE_MOVE STEPPER=stepper_z  DISTANCE=5 VELOCITY=5
     FORCE_MOVE STEPPER=stepper_z1 DISTANCE=5 VELOCITY=5
     FORCE_MOVE STEPPER=stepper_z2 DISTANCE=5 VELOCITY=5
     FORCE_MOVE STEPPER=stepper_z3 DISTANCE=5 VELOCITY=5
   {% endif %}
```

Call it at the head of **every** branch that moves X or Y — including the `G28 Z` branch,
which traverses to bed centre and would otherwise drag the nozzle across the plate. It only
ever goes up. Note it uses `FORCE_MOVE` on all four Z steppers rather than
`SET_KINEMATIC_POSITION` for the unhomed case: on this Klipper,
`SET_KINEMATIC_POSITION Z=0` marks **x, y and z** as homed even though you only passed `Z=`.
That leaves Y flagged homed at 0 after a cold `G28 X` while the gantry is physically at the
rear — jog Y from there and it drives against a false origin. `FORCE_MOVE` never touches
homed state. It moves the four motors one at a time, so the gantry sits skewed by up to
5 mm between them; the next `QUAD_GANTRY_LEVEL` corrects it. `[force_move]` with
`enable_force_move: True` is already in the stock `Macro.cfg`.

Second rule, and this is the one that will bite you: **`SET_KINEMATIC_POSITION Z=347` must
come before the traverse to bed centre, not after.**

```gcode
     {% if not z_known %}
       SET_KINEMATIC_POSITION Z=347
     {% endif %}
     G0  X191 Y165 F3600
     G28 Z
     G0  Z10 F600
```

`Z=347` fabricates a worst-case top so the probing move has the full travel available to
reach the bed. It authorizes a very long descent, so it is only safe over the plate — which
is why the traverse has to succeed first. And with Z unhomed, that traverse **is rejected**:

```
!! Must home axis first: 24.298 338.950 -0.019
```

A pure XY move shouldn't be checked against Z limits, but `FORCE_MOVE` and
`SET_KINEMATIC_POSITION` desync `gcode_move`'s last position from the kinematics, so the
move carries a residual Z delta of about 0.02 mm — enough to stop being a pure-XY move and
trip the unhomed-Z check. Put the fabrication first and the traverse is legal. Put it
second and the traverse is silently refused, the head stays wherever it was, and `G28 Z`
then descends 347 mm at that position. If that position is a corner, there is no plate
under the probe, nothing triggers, and **the nozzle is driven into the bed.** That is not
hypothetical — it is how this section got written.

### Verify

```bash
# from a genuine cold start
curl -s -X POST "http://<printer>:7125/printer/firmware_restart"
curl -s -X POST "http://<printer>:7125/printer/gcode/script?script=G28"
curl -s "http://<printer>:7125/printer/objects/query?toolhead=position,homed_axes" | python3 -m json.tool
```

`homed_axes` should read `"xyz"` and the head should end at bed centre. Then check the
single-axis case, which is where the false-homing bug shows:

```bash
curl -s -X POST "http://<printer>:7125/printer/firmware_restart"
curl -s -X POST "http://<printer>:7125/printer/gcode/script?script=G28%20X"
```

`homed_axes` must read `"x"` — not `"xyz"`. If it reads `"xyz"` you are still fabricating
coordinates with `SET_KINEMATIC_POSITION` somewhere in the lift path.

Repeated `G28`s from different starting heights should land Y on the same number every
time. Before the fix they don't. **Test high, not low** — raise the gantry to 200 mm or
more before homing, because that is where the tube pulls tight and where the false trigger
lives. A clean `G28` from Z 0 tells you nothing.

### Revert

Restore your backup of `printer.cfg` and `FIRMWARE_RESTART`. Take one *before* you start —
this fix is several edits deep and the intermediate states can drive the toolhead into the
bed.

---

## Does any of this apply to the Sovol SV08 Max?

Some of it. Sovol quietly fixed three of these on the Max — and carried the worst one forward.

**I do not own an SV08 Max.** Everything below is read out of [Sovol's own published source](https://github.com/Sovol3d/SV08MAX) — the stock `printer.cfg`, `plr.cfg`, `buffer_stepper.cfg`, `Macro.cfg`, `moonraker.conf`, and their patched `klippy/extras/gcode_move.py` — diffed against the SV08. It is a config-level read, not a hands-on one. Treat it as a map, not a measurement.

| # | On the SV08 Max |
|---|---|
| 4 — paused print goes cold | ✅ **Fixed.** The Max's `_IDLE_TIMEOUT` checks `printer.print_stats.state == "paused"` and skips `TURN_OFF_HEATERS`. The SV08's doesn't. |
| 9 — exhaust fan PID'd against the CPU | ✅ **Fixed.** The Max has **no `temperature_fan` at all**. The exhaust is a plain `[fan_generic fan3]` on `PE11`. The SV08's worst bug simply does not exist there. |
| 10 — filament sensor pauses on one read | ✅ **Fixed** — and Sovol landed on the same shape I did. The Max ships `pause_on_runout: False` with a custom `runout_gcode`. That makes the SV08's `pause_on_runout: True` the outlier, not the norm. (The Max's handler isn't a debounce — it calls `CONTINUE_PRINT_D` and keeps printing off the buffer — but the *architecture* is the same: don't let Klipper's built-in pause fire on a single read.) |
| 7 — PLR `os.fsync()` per move | ❌ **Still there — with the guard loosened.** See [Fix 7](#the-sv08-max-ships-the-same-patch--with-a-looser-guard). This is the one to carry across. |
| 8 — `M106 P` and the undocumented fans | ⚠️ **Still undocumented, and the map is different.** Max: `fan0` = front part fan (`extra_mcu:PB0`), `fan1` = rear part fan (`extra_mcu:PA7`), `fan2` = auxiliary (`PB0`), `fan3` = exhaust (`PE11`), plus `heater_fan hotend_fan` and a `heater_fan bed_fan` on `PE14`. **Do not reuse my P-numbers on a Max** — re-derive them from your own `printer.cfg`. |
| 5 — warped bed | ⚠️ Max owners report the same taco. Plus a Max-specific one I have no fix for: the bed **overshooting into an emergency shutdown**, climbing past target at 0% heater power. |
| 6 — `Timer too close` | ⚠️ Same platform, same eMMC, same PLR patch. Same three causes. Run the [free sysfs check](#check-yours-before-you-buy-anything). |
| 1, 2, 3 | Same Klipper/Moonraker/Mainsail stack. I did not verify these against the Max image. |

### No board cooling on the Max either

The Max's `printer.cfg` has **no mainboard fan** — no `output_pin`, no `heater_fan`, nothing pointed at the electronics. Same as a stock SV08, where the `PA1` header ships empty. A bigger machine with a bigger PSU and the same amount of board cooling: none. If you enclose it, [Fix 9](#fix-9--your-exhaust-fan-is-pid-controlled-by-a-cpu-it-isnt-pointed-at) is worth reading even though the PID bug it describes is gone.

### The Max answers the chamber-sensor question

The Max ships a `chamber_hot.cfg` (commented out by default): a **separate CAN toolboard** (`[mcu hot_mcu]`), two ordinary 100 K thermistors, a `[heater_generic chamber_temp]` capped at 65 °C on watermark control, and `M141`/`M191` macros.

Two things worth taking from that if you're retrofitting a chamber sensor to an SV08:

1. Sovol's own answer to "the CB1 has no spare GPIO" was **to add another MCU**, not to route sensors through the Linux host. That's also what the community does (a €5 Bluepill + I²C sensor).
2. They did **not** PID the exhaust against chamber temperature — the chamber heater is a separate `watermark` loop. Which is the right call: the target chamber temperature is filament-dependent (ABS wants it hot and the exhaust *off*; PLA and PETG want the opposite), so no single PID target serves both.

### The Obico beacon and the security holes are identical — the Obico one is worse

The Max ships the same `moonraker-obico.cfg`: `url = https://app.obico.io`, `sentry_opt = in`. And unlike the SV08, the Max's `printer.cfg` **includes `moonraker_obico_macros.cfg` on line 7, uncommented** — so the cloud plugin's macros are loaded into Klipper by default. [Fix 11](#fix-11--your-printer-announces-itself-to-a-cloud-service-you-never-signed-up-for) applies as-is.

The Max's `moonraker.conf` still binds `host: 0.0.0.0`, still lists all of RFC1918 under `trusted_clients`, and still has no `force_logins`. **Anyone on your LAN can move the gantry and start a print, with no credentials.** And [Sovol3d/SV08 issue #26](https://github.com/Sovol3d/SV08/issues/26) — *"SSH host keys not generated on first boot"* — is still open, with zero comments, two years on. Assume it's unfixed on the Max and run [Fix 12](#fix-12--ssh-host-keys-from-the-factory-and-passwordless-root) there too; it costs a minute and cannot break anything.

---

## Bonus: two things worth knowing

**Any Mainsail GUI setting lives in Moonraker's database**, namespace `mainsail` (keys: `console`, `control`, `dashboard`, `gcodeViewer`, `general`, `macros`, `miscellaneous`, `uiSettings`, `view`, …), readable and writable via `/server/database/item`. If you need a schema you don't know, grep the served JS bundle (`/assets/index-*.js`) rather than guessing.

**Moonraker's file-upload API has a trap.** The `path=` field is a **directory**, not a filename. `-F "path=aliases.cfg"` creates a *directory* called `aliases.cfg/` with your file inside it, and the next upload fails with `NotADirectoryError`. Put the name in the file part instead:

```bash
curl -X POST http://<printer-ip>:7125/server/files/upload \
  -F "root=config" \
  -F "file=@local.cfg;filename=aliases.cfg"
```

Clean up a bad one with `DELETE /server/files/directory?path=config/<name>&force=true`.

---

## Handy one-liners

```bash
# Live print state (no SSH needed)
curl -s "http://<printer-ip>:7125/printer/objects/query?print_stats&virtual_sdcard&display_status&idle_timeout"

# Moonraker version + klippy health + warnings
curl -s http://<printer-ip>:7125/server/info | python3 -m json.tool

# Timelapse settings
curl -s http://<printer-ip>:7125/machine/timelapse/settings | python3 -m json.tool

# Read any config file over HTTP
curl -s http://<printer-ip>:7125/server/files/config/printer.cfg
```

---

## Keywords

Sovol SV08 fixes · SV08 timelapse not working · SV08 timelapse auto render turns off after reboot · autorender resets · Mainsail Moonraker too old · update Moonraker to at least v0.8.0-306 · Moonraker version does not support all features of Mainsail · SV08 update_manager 404 · Moonraker v0.8.0-209 dirty · Klipper macro buttons G31 G34 M106 M600 · hide Mainsail macros · Mainsail hiddenMacros · Klipper rename macro breaks slicer · moonraker-timelapse blockedsettings · Sovol SV08 first setup · SV08 Klipper Mainsail Moonraker · SV08 pause turns off heaters · Klipper idle timeout during pause · printer cools down while paused · M600 filament change print ruined · pause too long print failed · Klipper idle_timeout 1800 · SET_IDLE_TIMEOUT pause not working · PAUSE defined twice Macro.cfg mainsail.cfg · _CLIENT_VARIABLE commented out · keep heaters on while paused Klipper · SV08 warped bed · Sovol SV08 bed not flat · SV08 v2 bed still warped · SV08 first layer inconsistent · SV08 bed mesh won't fix warp · SV08 corners lifting · bed mesh changes when heated · thermal bow heated bed · SV08 replacement bed · SV08 graphite bed · R3MEN graphite heated bed SV08 · does quad gantry level fix a warped bed · Klipper bed mesh vs flatness · SV08 Timer too close · Sovol SV08 MCU shutdown mid print · extra_mcu shutdown Timer too close · SV08 move queue overflow · Klipper move queue overflow organic supports · SV08 random shutdown long print · SV08 eMMC failure · replace SV08 eMMC module · eMMC pre_eol_info · eMMC life_time sysfs · mmcblk2 wear check · SV08 power loss recovery bug · sovol_plr_height · Sovol PLR os.fsync every move · Sovol3d SV08 issue 33 · SV08 Klipper shutdown not heat · SV08 EMI toolhead USB · SV08 dying flash · CB1 eMMC replacement · SV08 fan not working · SV08 M106 P1 both fans · Klipper M106 P parameter ignored · SV08 which fan is fan0 fan1 · SV08 part cooling fan front rear · SV08 fan runs at 0% · fan spinning but Mainsail shows 0 · SV08 hotend fan always on · heater_fan 45C Klipper · SV08 fan2 fan3 temperature_fan · Klipper fan stuck at 10 percent · min_speed 0.1 temperature_fan · SV08 exhaust fan header · SV08 fan_generic fan3 commented out · PA2 pin conflict SV08 · SET_FAN_SPEED vs M106 Klipper · SV08 fan rpm null · SV08 no fan tachometer · control SV08 fans individually · SV08 fan always at 100% · SV08 exhaust fan runs constantly · SV08 fan never turns off · SV08 loud fan idle · temperature_fan stuck at 100 percent · Klipper temperature_fan never reaches target · target_temp below idle temperature · SV08 temperature_host 65C · CB1 host temp high SV08 · SV08 host fan does not exist · SV08 PA2 exhaust fan · SV08 enclosure will not heat up · SV08 chamber temperature too low · SV08 ABS warping enclosure · enclosure never gets warm 3D printer · exhaust fan venting heated chamber · SV08 ASA printing enclosure · Klipper PID fan no authority · SV08 mainboard fan cools CB1 · SV08 fan2 target_temp 50 · SET_TEMPERATURE_FAN_TARGET 0 disables fan · Klipper temperature_fan target 0 turns fan off · SV08 M106 P3 exhaust fan working · SV08 chamber exhaust fan control · Klipper fan_generic off after reboot · Klipper fan_generic sticky between prints · Klipper fan resets to 0 on FIRMWARE_RESTART · OrcaSlicer air filtration SV08 · OrcaSlicer exhaust fan not working · support_air_filtration Orca · activate_air_filtration_during_print · OrcaSlicer ABS air filtration 70% default wrong · OrcaSlicer issue 9002 · PLA gcode has no M106 P3 · SV08 START_PRINT vs PRINT_START · plr.cfg PRINT_START never called · Klipper output_pin always on fan · Klipper shutdown_value fan · hide fan from Mainsail leading underscore · Klipper section name spaces not allowed · SV08 mainboard fan always 100% · SV08 temperature_sensor mcu_temp commented out · SV08 MCU temperature missing from Mainsail · SV08 exhaust throttles when host is cool · Klipper temperature_fan target at idle temp never spins up · SV08 pauses randomly mid print · SV08 false filament runout · Klipper filament sensor false trigger · filament_switch_sensor bounce · SV08 pauses with filament loaded · Pause Print! no reason Klipper · Klipper pause_on_runout no debounce · filament runout debounce Klipper · SV08 switch_pin PE9 · Klipper runout_gcode delayed_gcode · Klipper macro template rendered before execution · Klipper G4 dwell if statement stale · UPDATE_DELAYED_GCODE debounce · Klipper gcode_macro empty gcode variable holder · SV08 filament sensor chatter · pause_delay vs event_delay Klipper · SV08 host CPU over 70C during print · SV08 mainboard fan PA1 empty · Sovol SV08 Max fixes · does SV08 fix apply to SV08 Max · SV08 Max PLR fsync · SV08 Max os.fsync every move · SV08MAX gcode_move.py · SV08 Max power loss recovery bug · SV08 Max move queue overflow · SV08 Max Timer too close · SV08 Max fan map fan0 fan1 fan2 fan3 · SV08 Max exhaust fan PE11 · SV08 Max no temperature_fan · SV08 Max mainboard fan missing · SV08 Max board cooling · SV08 Max paused print heaters · SV08 Max idle_timeout paused · SV08 Max pause_on_runout False · SV08 Max filament sensor buffer_mcu · SV08 Max chamber heater chamber_hot.cfg · SV08 Max M141 M191 chamber · SV08 Max hot_mcu canbus · SV08 chamber temperature sensor retrofit · SV08 Max bed overtemp emergency shutdown · SV08 Max warped bed · Moonraker no authentication SV08 · force_logins Moonraker · Moonraker trusted_clients 0.0.0.0 · SV08 SSH host key not regenerated · Sovol SV08 issue 26 · SV08 MITM ssh host key · SV08 Timer too close heat vs eMMC vs EMI · SV08 obico default enabled · moonraker-obico auth_token not configured · Sovol printer phones home · app.obico.io unlinked · printer_discovery announce · SV08 telemetry sentry_opt · disable moonraker-obico · SV08 obico log 10MB · Sovol SV08 privacy · SV08 cloud connection default · self-host Obico SV08 · SV08 SSH host key factory · ssh_host_ed25519_key Dec 2023 · root@ubuntu host key · SV08 MITM · Sovol SV08 default password · SV08 passwordless sudo · SV08 NOPASSWD root · ssh-keygen -A SV08 · regenerate ssh host keys Sovol · REMOTE HOST IDENTIFICATION HAS CHANGED SV08 · SV08 security · Moonraker force_logins SV08 · SV08 unauthenticated Moonraker stock · SV08 park toolhead macro · SV08 maintenance position macro · Klipper park front center macro · Mainsail maintenance button · SV08 move toolhead to front · Klipper Y0 is front not Y_max · SV08 axis_maximum park · Klipper park at mid Z · TOOLHEAD_MAINTENANCE macro · SV08 nozzle access position · Klipper action_raise_error while printing

## License

MIT. Use at your own risk — you are editing a machine that gets hot and moves.
