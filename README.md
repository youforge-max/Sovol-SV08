# Fixes You Need After Buying a Sovol SV08

> Fixes you need after buying a Sovol SV08: warped bed (v1 & v2 — no DIY fix works), paused prints going cold and ruining the job, timelapse autorender resetting every boot, the Moonraker "too old" nag, and cryptic G-code macro buttons.

Seven things on a stock **Sovol SV08** that you will hit sooner or later:

1. **Timelapse auto-render turns itself off after every reboot.**
2. **Mainsail nags that Moonraker is too old** — and the updater is disabled.
3. **The macro dashboard is full of cryptic buttons** named `G31`, `G34`, `M106`, `M600`.
4. **A paused print goes cold after 30 minutes and is ruined.** ← this one costs you real prints
5. **The bed is warped** — on both the original bed and the "v2" revision. ← this one costs you *every* print
6. **Random `Timer too close` shutdowns mid-print.** Every forum will tell you it's heat or EMI. On my machine it was the **stock 8 GB eMMC** — swapping it fixed what nothing else would. ← this one costs you the board
7. **Power-loss recovery calls `os.fsync()` on every single move.** It kills prints with organic supports or z-hop *now*, and it is what makes a slow or worn eMMC (#6) fatal. **Do this one first — it's free.** ← this one costs you both

The first three are annoyances. **[Fix 4](#fix-4--a-paused-print-goes-cold-after-30-minutes-and-is-ruined) will destroy a 20-hour job** — pause for a filament change, get distracted, come back to a cold printer and a part that has let go of the bed. **[Fix 5](#fix-5--the-bed-is-warped--and-no-diy-fix-actually-works) is the one that quietly taxes every single print you make**, and it is the only one here whose real answer is "buy a new part".

**[Fix 6](#fix-6--timer-too-close-is-the-emmc-not-a-hot-mainboard) and [Fix 7](#fix-7--power-loss-recovery-writes-to-flash-on-every-single-move) are the same story told twice.** Sovol's power-loss-recovery patch forces a write to physical flash on every move; that stalls the host until the MCU gives up (`move queue overflow`, `Timer too close`), and it grinds the eMMC to death over months. I chased the `Timer too close` shutdowns through every forum thread I could find — heat, EMI, fans — and none of it worked. Replacing the eMMC did. Fix 7 is why it wore out.

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
| [6](#fix-6--timer-too-close-is-the-emmc-not-a-hot-mainboard) | `Timer too close` shutdowns — it's the eMMC, not heat | 💸 New eMMC | **Destroys prints + the board** |
| [7](#fix-7--power-loss-recovery-writes-to-flash-on-every-single-move) | PLR `os.fsync()` per move → `move queue overflow`, dead flash | Free | **Destroys prints + your eMMC** |

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

It also explains the two things heat can't:

* **Why it hits late in long prints.** More writes, more accumulated retries.
* **Why a fan sometimes "fixes" it.** It doesn't. Pulling the machine apart to mount a fan means reseating the eMMC — and a reseated (or replaced) module is the actual change.

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

Sovol SV08 fixes · SV08 timelapse not working · SV08 timelapse auto render turns off after reboot · autorender resets · Mainsail Moonraker too old · update Moonraker to at least v0.8.0-306 · Moonraker version does not support all features of Mainsail · SV08 update_manager 404 · Moonraker v0.8.0-209 dirty · Klipper macro buttons G31 G34 M106 M600 · hide Mainsail macros · Mainsail hiddenMacros · Klipper rename macro breaks slicer · moonraker-timelapse blockedsettings · Sovol SV08 first setup · SV08 Klipper Mainsail Moonraker · SV08 pause turns off heaters · Klipper idle timeout during pause · printer cools down while paused · M600 filament change print ruined · pause too long print failed · Klipper idle_timeout 1800 · SET_IDLE_TIMEOUT pause not working · PAUSE defined twice Macro.cfg mainsail.cfg · _CLIENT_VARIABLE commented out · keep heaters on while paused Klipper · SV08 warped bed · Sovol SV08 bed not flat · SV08 v2 bed still warped · SV08 first layer inconsistent · SV08 bed mesh won't fix warp · SV08 corners lifting · bed mesh changes when heated · thermal bow heated bed · SV08 replacement bed · SV08 graphite bed · R3MEN graphite heated bed SV08 · does quad gantry level fix a warped bed · Klipper bed mesh vs flatness · SV08 Timer too close · Sovol SV08 MCU shutdown mid print · extra_mcu shutdown Timer too close · SV08 move queue overflow · Klipper move queue overflow organic supports · SV08 random shutdown long print · SV08 eMMC failure · replace SV08 eMMC module · eMMC pre_eol_info · eMMC life_time sysfs · mmcblk2 wear check · SV08 power loss recovery bug · sovol_plr_height · Sovol PLR os.fsync every move · Sovol3d SV08 issue 33 · SV08 Klipper shutdown not heat · SV08 EMI toolhead USB · SV08 dying flash · CB1 eMMC replacement

## License

MIT. Use at your own risk — you are editing a machine that gets hot and moves.
