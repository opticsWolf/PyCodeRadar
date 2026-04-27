
<img width="256" height="256" alt="grafik" src="https://github.com/user-attachments/assets/db22026e-1eb3-4d8e-9298-684e9118316d" /> 

# PyCodeRadar

## How to run

Either of these still works:

```
python PyCodeRadar.py
python -m pycoderadar
```

`PyCodeRadar.py` is now a 5-line shim that calls `pycoderadar.app.main`.

## Module layout

```
pycoderadar/
├── __init__.py
├── __main__.py         # `python -m pycoderadar`
├── app.py              # main() — QApplication + palette + show window
├── constants.py        # tool-availability flags, EXCLUDE_DIRS, app identity
├── models.py           # @dataclass: ImportInfo, FunctionInfo, ScanOptions, …
├── presets.py          # NEW — strictness presets + detect_preset()
├── config.py           # NEW — load / save JSON config via QStandardPaths
├── analysis.py         # AST helpers, CodeMapVisitor, build_module_map
├── external_tools.py   # Ruff / Radon / Mypy runners + build_mypy_cmd
├── formatters.py       # to_text / to_json
├── option_defs.py      # TRACK / ANTIPATTERN / LEGACY / MYPY_STRICT_FLAGS metadata
├── styles.py           # dark Qt stylesheet
├── widgets.py          # FileTreeWidget + MapHighlighter
├── worker.py           # ScanWorker (QThread)
└── main_window.py      # MainWindow — owns the GUI
```

## Strictness presets

The dropdown sits at the top of the options panel. Picking a preset rewrites
**every** option below it. Manually changing any option flips the dropdown to
**Custom** automatically — it's a status indicator, not just a menu choice.

| Preset    | Tracking | Anti-patterns | Legacy | Mypy strict | Thresholds |
|-----------|----------|---------------|--------|-------------|------------|
| Lenient   | minimal  | only critical | off    | off         | generous (100 / 8 / 25) |
| Standard  | sensible | most          | off    | per-flag    | default  (50 / 5 / 15) |
| Strict    | full     | full + unused | all    | `--strict`  | tighter  (40 / 4 / 12) |
| Pedantic  | full     | + `print()`   | all    | `--strict` + show notes | strictest (30 / 3 / 10) |
| Custom    | — set automatically when you tweak any individual checkbox or threshold |

Preset definitions live in `pycoderadar/presets.py`. To add or tune a preset,
edit the `_PRESETS` dict there — no GUI changes needed.

## Persisted config

The config file is written to the OS-appropriate per-user location
(via `QStandardPaths`):

- **Linux**:   `~/.config/PyCodeRadar/config.json`
- **macOS**:   `~/Library/Preferences/PyCodeRadar/config.json`
- **Windows**: `%APPDATA%\PyCodeRadar\config.json`

It's plain JSON — safe to edit by hand or delete to reset everything:

```json
{
  "version": 1,
  "preset": "Standard",
  "format": "Text",
  "last_folder": "/path/to/last/project",
  "scan_options": { "...": "..." },
  "window": {
    "geometry": "<base64 QByteArray>",
    "splitter": [320, 880]
  }
}
```

Saved on every window close (including via window-manager quit).
Loaded once at startup; missing keys fall back to defaults, unknown keys are
ignored — old configs continue to work after upgrades.

## Active linting

Until now PyCodeRadar's external-tool integration was strictly read-only —
issues were reported but never modified. The new **🔧 Fix** button next to
**▶ Analyse** runs Ruff in active mode against the selected files.

### LINT & FIX card

A new card in the options panel exposes four toggles:

| Option | What it does |
|---|---|
| **Apply Ruff auto-fixes** | `ruff check --fix` — strips unused imports, sorts imports, removes unreachable code, etc. |
| **Include unsafe fixes** | adds `--unsafe-fixes` for rules Ruff classifies as potentially semantics-changing |
| **Reformat code** | `ruff format` (Black-compatible) |
| **Preview only** | uses `--diff`: shows what *would* change, writes nothing — **on by default** |

Pick at least one of the first three actions, then hit **🔧 Fix**. The diff
renders inline with proper colors (green `+`, red `-`, cyan `@@` markers).

### Safety rails

- **Preview-only is the default.** First-time use shows a diff; nothing is
  written until you uncheck it.
- **Confirmation dialog** when committing — lists the action(s) and file count.
- **Fix button is disabled** unless Ruff is installed AND a folder is loaded
  AND files are selected AND at least one action is checked.
- **Re-analyse hint** in the status bar after a successful apply, so the
  report can be refreshed.

### After the fact

Lint settings are persisted alongside everything else. The toggles also
participate in the preset / Custom logic — flipping any of them flips the
preset combo to **Custom** so your choice survives the next session intact.
