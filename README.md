# Fixes You Need After Buying a Sovol SV08

Three things on a stock **Sovol SV08** that quietly annoy you until you fix them: **timelapse auto-render turns itself off after every reboot**, **Mainsail nags that Moonraker is too old**, and the **macro dashboard is full of cryptic buttons named `G31`, `G34`, `M106`, `M600`**.

None of them stop the printer working. All three are a five-minute fix. This is what causes them and how to fix each one properly — including *why* the obvious fix is wrong in two of the three cases.

Tested on an SV08 running the factory Sovol image (Klipper + Moonraker + Mainsail, Moonraker `v0.8.0-209-g4235789-dirty`). Nothing here is SV08-exclusive — the timelapse bug hits any `moonraker-timelapse` install, and the macro-button problem hits any Klipper printer whose vendor named macros after raw G-code.

> **Back up before you touch anything.** `cp file file.bak.$(date +%Y%m%d)`. Every fix below tells you how to revert it.

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

Sovol SV08 fixes · SV08 timelapse not working · SV08 timelapse auto render turns off after reboot · autorender resets · Mainsail Moonraker too old · update Moonraker to at least v0.8.0-306 · Moonraker version does not support all features of Mainsail · SV08 update_manager 404 · Moonraker v0.8.0-209 dirty · Klipper macro buttons G31 G34 M106 M600 · hide Mainsail macros · Mainsail hiddenMacros · Klipper rename macro breaks slicer · moonraker-timelapse blockedsettings · Sovol SV08 first setup · SV08 Klipper Mainsail Moonraker

## License

MIT. Use at your own risk — you are editing a machine that gets hot and moves.
