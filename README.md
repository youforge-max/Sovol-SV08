# Fixes You Need After Buying a Sovol SV08

Five things on a stock **Sovol SV08** that you will hit sooner or later:

1. **Timelapse auto-render turns itself off after every reboot.**
2. **Mainsail nags that Moonraker is too old** — and the updater is disabled.
3. **The macro dashboard is full of cryptic buttons** named `G31`, `G34`, `M106`, `M600`.
4. **A paused print goes cold after 30 minutes and is ruined.** ← this one costs you real prints
5. **The bed is warped** — on both the original bed and the "v2" revision. ← this one costs you *every* print

The first three are annoyances. **[Fix 4](#fix-4--a-paused-print-goes-cold-after-30-minutes-and-is-ruined) will destroy a 20-hour job** — pause for a filament change, get distracted, come back to a cold printer and a part that has let go of the bed. **[Fix 5](#fix-5--the-bed-is-warped--and-no-diy-fix-actually-works) is the one that quietly taxes every single print you make**, and it is the only one here whose real answer is "buy a new part".

Each fix below covers what causes it, how to fix it, how to verify it, and how to revert — including *why the obvious fix is wrong* in most of them.

Tested on an SV08 running the factory Sovol image (Klipper + Moonraker + Mainsail, Moonraker `v0.8.0-209-g4235789-dirty`). Fixes 1–4 are software and cost nothing; **Fix 5 is hardware and costs money** — it is last precisely because it is the one you should be slowest to accept.

Most of this isn't even SV08-exclusive: the timelapse bug hits any `moonraker-timelapse` install, the macro-button problem hits any Klipper printer whose vendor named macros after raw G-code, and the paused-print cool-down hits any Klipper machine with a stock `idle_timeout`. The warped bed is the SV08-specific one.

> **Back up before you touch anything.** `cp file file.bak.$(date +%Y%m%d)`. Every fix below tells you how to revert it.

| # | Problem | Cost | Severity |
|---|---|---|---|
| [1](#fix-1--timelapse-auto-render-resets-to-off-after-every-reboot) | Timelapse auto-render resets to OFF on every reboot | Free | Annoyance |
| [2](#fix-2--mainsail-moonraker-too-old-update-to-at-least-v080-306) | Mainsail: "Moonraker too old" + updater disabled | Free | Cosmetic |
| [3](#fix-3--mainsail-dashboard-is-full-of-g31--g34--m106--m600-buttons) | Dashboard full of raw G-code buttons | Free | Annoyance |
| [4](#fix-4--a-paused-print-goes-cold-after-30-minutes-and-is-ruined) | Paused print cools down and is ruined | Free | **Destroys prints** |
| [5](#fix-5--the-bed-is-warped--and-no-diy-fix-actually-works) | Warped bed (v1 **and** v2) — no DIY fix works | 💸 New bed | **Degrades every print** |

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

Sovol SV08 fixes · SV08 timelapse not working · SV08 timelapse auto render turns off after reboot · autorender resets · Mainsail Moonraker too old · update Moonraker to at least v0.8.0-306 · Moonraker version does not support all features of Mainsail · SV08 update_manager 404 · Moonraker v0.8.0-209 dirty · Klipper macro buttons G31 G34 M106 M600 · hide Mainsail macros · Mainsail hiddenMacros · Klipper rename macro breaks slicer · moonraker-timelapse blockedsettings · Sovol SV08 first setup · SV08 Klipper Mainsail Moonraker · SV08 pause turns off heaters · Klipper idle timeout during pause · printer cools down while paused · M600 filament change print ruined · pause too long print failed · Klipper idle_timeout 1800 · SET_IDLE_TIMEOUT pause not working · PAUSE defined twice Macro.cfg mainsail.cfg · _CLIENT_VARIABLE commented out · keep heaters on while paused Klipper · SV08 warped bed · Sovol SV08 bed not flat · SV08 v2 bed still warped · SV08 first layer inconsistent · SV08 bed mesh won't fix warp · SV08 corners lifting · bed mesh changes when heated · thermal bow heated bed · SV08 replacement bed · SV08 graphite bed · R3MEN graphite heated bed SV08 · does quad gantry level fix a warped bed · Klipper bed mesh vs flatness

## License

MIT. Use at your own risk — you are editing a machine that gets hot and moves.
