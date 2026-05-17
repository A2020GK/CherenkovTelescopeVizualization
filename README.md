# CherenkovTelescopeVizualization

PyQt6-based visualization tool for the HiSCORE Cherenkov telescope array. The app renders the station layout, lists events from an event file, and opens per‑station waveform views when you click a station.

## Repository layout

- `HiSCORE/hiscore.py` — main application entry point (GUI).
- `HiSCORE/c.dat` — station geometry used to position buttons.
- `HiSCORE/hiconfig.ini` — persisted UI settings and last opened paths.
- `HiSCORE/Data/` — sample data (`hiscore.draw`, `hiscore.test`, `station_XX.dat`).
- `HiSCORE/requirements.txt` — Python dependencies.

## Setup & run

From the `HiSCORE/` directory:

1. (Optional) Create and activate a virtual environment.
2. Install system dependency: `libxcb-cursor0` (required by PyQt on Linux).
3. Install Python dependencies: `pip install -r requirements.txt`
4. Run: `python3 hiscore.py`

The app can open a data directory and an event list file via **File → Open Directory** and **File → Open file**. The last used paths are stored in `hiconfig.ini`.

## How the app uses data

The UI is data‑driven. The station geometry determines where buttons appear. The event list file determines which stations are active for a given event and the numeric value used for coloring. Per‑station waveform files provide the detailed signals shown in the station window.

### 1) Station geometry (`c.dat`)

**Format:** whitespace‑separated, no header.  
**Columns (in order):**

1. `ch` — station/channel ID.
2. `x` — X coordinate.
3. `y` — Y coordinate.
4. `z` — Z coordinate (stored but not used in plotting).

**Usage in code:**

- Each row becomes a station button.
- Button placement uses `x` and `y` (internally swapped and negated).
- `ch` determines the “SiPM group”:
  - `< 100` → group 1
  - `< 200` → group 2
  - `< 300` → group 3
  - `>= 300` → group 4

### 2) Event list file (`.draw` / `.test`)

**Selected via:** **File → Open file** or `hiconfig.ini` (`Set_path.filename`).  
**Format:** a sequence of event blocks.

Each block begins with a **header line** (no leading spaces) containing a single integer `N` — the number of station records for that event. The next `N` lines (usually indented) are the station records.

**Station record fields (0‑based index):**

0. `station_id` — matches `c.dat` `ch` and the station file name (`station_XX.dat`).
1. `station_event_id` — event ID used to locate the event inside the station file.
2. `event_time` — displayed in the event table.
3. `field3` — present but unused by the app.
4. `field4` — present but unused by the app.
5. `value` — numeric value used for station coloring and text.

**Usage in code:**

- The event table columns **st / st_event / st_time** are populated from fields 0–2.
- Field 5 (`value`) drives button color and the numeric label shown on the station.
- The event count determines the contents of the event selector combobox.

### 3) Station waveform files (`station_XX.dat`)

**Selected via:** **File → Open Directory** or `hiconfig.ini` (`Set_path.directory`).  
**Naming:** `station_01.dat`, `station_02.dat`, …  
**Format:** whitespace‑separated, **401 columns**, no header. Columns are named `"0"`…`"400"` in the code.

Each event occupies **10 consecutive rows**:

1. **Event header row** (column `0` is a numeric event ID).
2. 9 **signal rows** with labels in column `0`:
   - `a1`, `a2`, `a3`, `a4` — anode signals
   - `d1`, `d2`, `d3`, `d4` — diode signals
   - `tr1` — trigger/aux signal

**Event header columns (0‑based):**

0. `event_id` — matched against `station_event_id` from the event list file.
1. `timestamp` — shown in the station window title.
2. `field2` — unused.
3. `field3` — unused.
4. `min` — start time index for plotting.
5. `dlin` — signal length (used to compute the X‑axis range).
6–400 — unused for the header row.

**Signal rows:**

- Column `0` is the label (`a1`, `a2`, …).
- Columns `1..400` are integer samples (400 points).
- The app plots samples on the X‑axis `range(min, min + dlin)` and lets you trim the view with spin boxes.

### 4) Configuration (`hiconfig.ini`)

The app reads settings from `hiconfig.ini` at startup. The most important section for data loading is:

```
[Set_path]
directory = /path/to/station/files
filename = /path/to/event/list/file
```

If these paths exist, the app auto‑loads them on launch. Other sections control layout and UI defaults.

## UI behavior (high‑level)

- **Main window** shows the station layout and an event selector.
- **Event selector** moves through event blocks defined in the event list file.
- **Station buttons** show the station ID and the value from the event list record; color is based on the same value.
- **Clicking a station** opens a station window with three plots:
  - Anode (`a1`–`a4`)
  - Diode (`d1`–`d4`)
  - Trigger (`tr1`)
