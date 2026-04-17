# OpenHaC — Open Hardware-as-Code

A Python compiler that turns declarative hardware code into manufacturable PCB designs, **netlists**, optional **routing** (FreeRouting), and SPICE simulations — no GUI required.

## What it does

Write your hardware design in Python. Define components, wire modules together, set spatial constraints, and the compiler produces:

- `.net` + `.csv` — SKiDL netlist and BOM (LCSC-oriented fields when available)
- `.kicad_pcb` — KiCad board with outline, **footprint placement**, and pad nets (optional **autoroute** via FreeRouting)
- `.kicad_sch` / `.kicad_pro` — optional schematic + project stub (when `export_schematic=True`)
- `sym-lib-table` + `*.openhac-generated.kicad_sym` — project-local symbol library for SKiDL-native parts (prevents `?` symbols in KiCad)
- `.openhac-manifest.json` — build output inventory (after a successful `compile`; optional `output_dir` / `openhac compile -o DIR` bundles artifacts — **MFG-005** / **STR-002**)
- `*.openhac-evidence.md` — evidence index for review/sign-off (checks run, key metrics)
- `*.openhac-attestation.json` — optional attestation metadata (when enabled)
- `*.openhac-sipi-handoff.json` — consolidated SI/PI intent handoff bundle
- `.cir` — ngspice-oriented SPICE netlist from `Board.simulate()`

**Scope:** See [docs/SCOPE.md](docs/SCOPE.md) for capability tiers and non-goals.  
**Release engineering:** See [docs/RELEASE_CHECKLIST.md](docs/RELEASE_CHECKLIST.md) and [docs/IMPLEMENTATION_STATUS.md](docs/IMPLEMENTATION_STATUS.md) for spec tracking.

**Tier C / autorouter honesty (PCB-007 / SIG-002):** FreeRouting is **placement/routing assistance**. It is **not** a substitute for manual **high-speed** routing, **differential-pair** impedance control, or **EMC** review — `Board.route_differential_pair()` is stored as a constraint but **not** consumed by the placer/router; controlled pairs must be finished in KiCad.

**PCB fab-intent (keepouts / net-ties / pours):**
- **Keepouts**: `Board.declare_keepout_rect(...)` emits a best-effort pcbnew **rule-area keepout** rectangle into the generated `.kicad_pcb` (when `pcbnew` is available). Use this for antenna zones, edge connector clearance, and “no copper/placement” regions.
- **Net-ties**: `Board.declare_net_tie(net_a, net_b, ...)` emits a **NetTie** footprint into the `.kicad_pcb` with pad-1/pad-2 assigned to each net (when footprint libraries are available). `Board.declare_net_merge_hint(..., via="net_tie")` also records a net-tie intent automatically (SIG-006 stretch).
- **Copper pours**: `Board.declare_copper_pour_intent(...)` emits best-effort rectangular copper zones on requested layers (when `pcbnew` is available). Always review/finalize zones and clearances in KiCad before fab.

**CLI:** `openhac --version` prints the installed package version (aligned with HTTP User-Agent).

The pipeline uses SKiDL, KiCad (`pcbnew` / `kicad-cli`), optional FreeRouting, and ngspice-style SPICE.

---

## Architecture

**Tier 1 — Component Database (`openhac/database/`)**

SQLite-backed catalog of real, orderable components. Every `Component("name")` call resolves to a real MPN, LCSC supplier SKU, KiCad symbol, and footprint.

- `sync_catalog()` fetches live in-stock parts from the [jlcsearch API](https://jlcsearch.tscircuit.com) (backed by the JLCPCB catalog) across 9 categories: resistors, capacitors, LEDs, MOSFETs, microcontrollers, voltage regulators, diodes, switches, accelerometers
- Components not in the local DB are resolved via live LCSC API lookup and cached automatically — no manual entry needed
- `db.search_components(query, category)` for parametric search

**Tier 2 — Core Abstraction (`openhac/core/`, `openhac/stdlib/`)**

- `Component(generic_name)` — resolves a part name to a real SKiDL `Part` with MPN/footprint injected from the DB
- `Module` — groups components, declares named `Interface` connection points, tracks power budget (`max_current_draw_ma`, `source_current_max_ma`)
- `Board` — connects modules, applies spatial constraints, drives the full compiler pipeline
- `stdlib/` — pre-wired module classes for common parts (ESP32-WROOM, LDO regulators, XT60 connectors, passives)

**Tier 3 — Compiler Pipeline (`openhac/compiler/`)**

- **ERC** (`rule_check.py`): floating nets, unconnected pins, missing power flags (prefix + optional `Board.declare_power_rail`), user `register_erc_hook` plugins (SCH-005; see `openhac.stdlib.erc_rules` for an I2C pull-up example), power budget overload
- **Interface validation**: all declared module interfaces must be connected before netlist generation proceeds
- **Netlist** (`netlist_gen.py`): SKiDL → `.net` + BOM `.csv` (**JLC_Class**, optional **Mouser_SKU** / **DigiKey_SKU** from DB — **LIB-001**)
- **Layout** (`layout_gen.py`, `pcb_placement.py`): Z3 module placement → KiCad board outline + footprint instances with pad nets; **`pin_pad_coverage_warnings`** compares SKiDL pin numbers to **`*.kicad_mod`** pads before `SetNet` (**PCB-002** diagnostic)
- **Autorouter** (`autoroute_cli.py`): FreeRouting jar via subprocess, DSN/SES workflow (not HS-aware; see **PCB-007** in [docs/SCOPE.md](docs/SCOPE.md))
- **Schematic** (`schematic_gen.py`): KiCad S-expression `.kicad_sch` with symbol instances and wire geometry; **`schematic_geometry`** + parsers support round-trip checks vs on-disk wires/labels (**SCH-001**)
- **SPICE** (`spice_gen.py`): `.cir` netlist using `Part.ref_prefix` for correct SPICE element identifiers

---

## Installation

```bash
pip install -e .
# optional: pytest, ruff, mypy (CI / contributors)
pip install -e ".[dev]"
```

Dependencies are declared in **`pyproject.toml`** (no separate `requirements.txt`).

**Requirements:**
- Python 3.11+
- KiCad 7 or 8 with Python bindings (for PCB/schematic output)
- For PCB layout generation, OpenHaC needs the **`pcbnew` Python module** (KiCad Python bindings), not just a `pcbnew` binary on `PATH`.
- KiCad footprint libraries on disk — set **`KICAD8_FOOTPRINT_DIR`** (or **`KICAD9_FOOTPRINT_DIR`**) to the folder that contains `*.pretty` directories (e.g. `/usr/share/kicad/footprints` on Linux) so the compiler can place footprints
- Java runtime (for FreeRouting autorouter)

---

## Environment variables

| Variable | Purpose |
|----------|---------|
| **`OPENHAC_DB_PATH`** | Path to the SQLite component catalog (default: `openhac/database/openhac.db` inside the install). Use a writable path for CI or multi-project isolation. |
| **`OPENHAC_SKIP_LAYOUT`** | If `1` / `true` / `yes`, `Board.compile()` skips KiCad `pcbnew` layout generation and autoroute — emits **`.net`**, **`.csv`**, and manifest only (headless CI / logic-only builds). |
| **`OPENHAC_COMPILE_GOAL`** | Compile gate: `handoff` (reviewable KiCad outputs; best-effort routing allowed) vs `fabrication` (fail-closed gates like PCB fit + requiring a production router). |
| **`OPENHAC_DETERMINISTIC`** | If set, prefer byte-stable outputs (stable zip entries, stable schematic UUIDs/ordering, stable manifest fields) for golden/CI artifact comparisons. |
| **`OPENHAC_STRICT_JIT`** | If set, medium-confidence live/JIT part lookups are treated like low-confidence unless risky lookups are explicitly allowed (see `openhac compile --strict-jit`). |
| **`OPENHAC_ALLOW_RISKY_PARTS`** | Allows low/medium JIT mappings when strict JIT is on (escape hatch; prefer an explicit DB seed/sync for production). |
| **`OPENHAC_REQUIRE_VERIFIED_PARTS`** | If set, DRC fails when any **medium/low** confidence JIT parts are present (production gate). |
| **`OPENHAC_KICAD_SYMBOL_DIRS`** | Pathsep-separated extra symbol search dirs for SCH-001 pin-position lookup (prepended ahead of KiCad defaults). |
| **`OPENHAC_SCHEMATIC_MULTI_SHEET`** | If set, schematic export emits a root sheet + one subsheet per module, with sheet pins + child hierarchical labels derived from `Module.declare_interface()` nets. |
| **`OPENHAC_ATTEST_SIGNER`** | If set, OpenHaC emits `*.openhac-attestation.json` metadata with `signer=<value>` (signing is future; metadata is useful now). |
| **`OPENHAC_RELEASE_TAG`** | Optional string recorded in **``release_tag``** on the compile manifest (STR-002); CLI **``--release-tag``** overrides for one run. |
| **`OPENHAC_BUILD_PROFILE`** | Optional string recorded as **``build_profile``** in the manifest (e.g. `production`). |

Fabrication export also relies on **`KICAD8_FOOTPRINT_DIR`** / **`KICAD9_FOOTPRINT_DIR`** (or **`KICAD_FOOTPRINT_DIR`**) and KiCad symbol paths as documented by KiCad for your OS.

---

## Production readiness

Normative list: **[docs/PRODUCTION_READINESS_SPEC.md](docs/PRODUCTION_READINESS_SPEC.md)**.  
Status (all **48** spec IDs **Done** under Phase-1): **[docs/IMPLEMENTATION_STATUS.md](docs/IMPLEMENTATION_STATUS.md)**.

**Phase-1 completion** means each ID meets narrowed acceptance (shipped behavior + handoff + tests/docs) as described in the spec’s **Phase-1 completion** section. Per-ID **Notes** still call out **stretch / future** work (e.g. multi-sheet KiCad, in-tool SI, release signing) where aspirational **Target state** text in the spec describes a longer horizon.

**CI:** The main workflow runs Ruff + pytest on Python 3.11/3.12. An optional job **`kicad-layout-smoke`** installs KiCad on Ubuntu and runs **`scripts/ci_full_compile_smoke.py`** (full **`.kicad_pcb`** when `pcbnew` imports); it is **`continue-on-error`** so KiCad/apt drift does not block merges.

---

## LaTeX report

A hand-maintained, detailed LaTeX report lives under `docs/report/`.

Build the PDF (requires a LaTeX distribution providing `lualatex` or `pdflatex`):

```bash
python3 scripts/build_latex_report.py
```

## Setup

### 1. Sync the component database

Pull ~1,400 real in-stock JLCPCB components into the local SQLite cache:

```bash
python -m openhac.database.sync_jlc
```

This is a one-time operation. Re-run periodically to pick up new parts. Any component not in the cache is resolved automatically via live lookup when first used.

### 2. (Optional) Seed baseline parts

The seed script pre-loads a small set of hand-verified parts (ESP32-WROOM, AMS1117, XT60, common passives):

```bash
python -m openhac.database.seed_data
```

---

## Usage

### Defining a Module

A `Module` groups related components and exposes named `Interface` connection points. Internal pin wiring stays private; external connections go through interfaces.

```python
from openhac.core.base import Module, Component
from skidl import Net

class MyMCU(Module):
    def __init__(self):
        super().__init__("MyMCU")

        self.ic = self.add(Component("ESP32_WROOM"))

        vcc = Net("3V3")
        gnd = Net("GND")

        # Internal wiring — raw pin access is fine inside __init__
        self.ic['2'] += vcc
        self.ic['1'] += gnd

        # Declare the external interface — this is what other modules connect to
        self.power = self.declare_interface("power", vcc, gnd)
```

Accessing a module's internal pins from outside its `__init__` emits a `DeprecationWarning` — use `expose_interface()` instead.

### Using stdlib Modules

Pre-wired modules are ready to use directly:

```python
from openhac.stdlib.mcu import ESP32_WROOM
from openhac.stdlib.power import XT60_Input, LDO_5V
from openhac.stdlib.passives import Resistor, Capacitor

mcu   = ESP32_WROOM()   # exposes: mcu.power (vcc, gnd)
ldo   = LDO_5V()        # exposes: ldo.v_in, ldo.v_out
power = XT60_Input()    # exposes: power.v_out

r = Resistor(value="10k", package="0805")
c = Capacitor(value="100nF", package="0603")
```

### Building a Board

```python
from openhac.core import Board
from openhac.stdlib.mcu import ESP32_WROOM
from openhac.stdlib.power import XT60_Input, LDO_5V

board = Board(size_mm=(60, 40), layers=2)

power = XT60_Input()
ldo   = LDO_5V()
mcu   = ESP32_WROOM()

board.add_module(power)
board.add_module(ldo)
board.add_module(mcu)

# Connect interfaces — no raw pin numbers needed at the board level
board.connect(power.v_out, ldo.v_in)
board.connect(ldo.v_out,   mcu.power)
```

### Spatial Constraints

```python
# Keep the power connector on the board edge
board.constrain_edge(power, "TOP")

# Regulator must be at least 8mm from the MCU (thermal isolation)
board.constrain_distance_min(ldo, mcu, 8)

# But no more than 15mm away (trace length)
board.constrain_distance_max(ldo, mcu, 15)
```

### Power Budget

```python
ldo.source_current_max_ma = 500   # LDO can supply 500mA
mcu.max_current_draw_ma   = 250   # ESP32 draws 250mA peak

# If total draw exceeds total supply, ERC raises ERCPowerBudgetError at compile time
```

### DRC trace width (IPC-2152)

If you set **`max_current_draw_ma`** on modules, DRC compares the IPC-2152 external-layer width to the design minimum (default **0.15mm**). Raise the board minimum when your fab rules allow wider default traces:

```python
board.min_trace_width_mm = 0.35  # e.g. heavy 3V3 rail
```

### Compiling

From Python:

```python
board.compile(
    project_name="my_board",   # output file prefix
    generate_bom=True,         # write my_board.csv
    auto_route=True,           # invoke FreeRouting (requires FREEROUTING_JAR env var)
    export_schematic=True,     # write my_board.kicad_sch + my_board.kicad_pro
)
```

Or use the CLI (requires a top-level variable named `board`; do not call `board.compile()` at import time when using this):

```bash
openhac compile my_design.py --name my_board
openhac compile my_design.py --no-route --no-schematic
openhac compile my_design.py --compile-goal fabrication   # stricter gates (fit checks; requires production router)
openhac compile my_design.py --strict-jit    # reject medium-confidence JIT unless risky parts allowed
openhac compile my_design.py --production     # strict KiCad symbols + strict JIT for this compile (LIB-004 / LIB-003)
openhac compile my_design.py --require-verified-parts   # fail if any medium/low confidence JIT parts are present
openhac compile my_design.py --skip-layout --deterministic -o out/    # headless, byte-stable artifacts
openhac compile my_design.py -o dist/rel --release-tag v1.0.0 --build-profile production --zip-release
openhac compile my_design.py --kicad-erc --kicad-erc-json   # JSON ERC report for parsing (SCH-003)
```

### Stress-test example (`flight_controller.py`)

This repo includes a deliberately large circuit definition (`flight_controller.py`) intended to stress OpenHaC’s netlist/BOM/schematic/layout pipeline.

Compile it into `out/` (project opens in KiCad, includes a PCB + schematic):

```bash
python3 -m openhac doctor --strict-layout
python3 -m openhac compile flight_controller.py --name fc_stress --deterministic -o out/
```

Notes:
- If **FreeRouting** is installed and `FREEROUTING_JAR` is set, OpenHaC will use it for autorouting.
- If `FREEROUTING_JAR` is not set but `pcbnew` imports, OpenHaC falls back to a **minimal pcbnew-based router** that adds a small number of tracks so the output PCB is not “empty”.
- Open the result via `out/fc_stress.kicad_pro` in KiCad.
- When `OPENHAC_COMPILE_GOAL=fabrication`, OpenHaC will also run **KiCad PCB DRC** (`kicad-cli pcb drc`) and fail closed on violations.

With **`--skip-layout`** (or **`OPENHAC_SKIP_LAYOUT=1`**), the same CLI can produce netlist + BOM + manifest without `pcbnew` (useful in CI). Use **`--deterministic`** (or env **`OPENHAC_DETERMINISTIC=1`**) for stable artifacts suitable for golden comparisons. Use **`--zip-release`** (optional **`--zip-release-path`**) to archive emitted artifacts.

Toolchain/config preflight:

```bash
openhac doctor --json
openhac doctor --strict-headless --json   # require kicad-cli + symbol config
openhac doctor --strict-layout --json     # require pcbnew + footprint config
openhac doctor --strict-config --json     # require lib table or KICAD*_DIR hints only
openhac doctor --strict-routing --json    # require java + FREEROUTING_JAR (autorouter)
openhac doctor --print-env                # print best-effort export lines
```

The `doctor --json` payload also includes a `kicad_env` map (relevant `KICAD*_SYMBOL_DIR` / `KICAD*_FOOTPRINT_DIR` vars plus `OPENHAC_KICAD_SYMBOL_DIRS`) and the compile manifest records the same map under `kicad_env` for audit/debug.

**FreeRouting autorouter** requires the jar path:

```bash
export FREEROUTING_JAR=/path/to/freerouting.jar
```

Or pass it directly:

```python
from openhac.compiler.autoroute_cli import run_freerouting
run_freerouting("my_board.kicad_pcb", freerouting_jar_path="/path/to/freerouting.jar")
```

### Fabrication export (Gerbers / drill / placement)

After you have a `.kicad_pcb` (from `board.compile` or KiCad), generate fab outputs with **KiCad’s** `kicad-cli` (must be on `PATH`):

```bash
openhac export fab my_board.kicad_pcb -o ./gerbers
openhac export fab my_board.kicad_pcb -o ./gerbers --no-pos
openhac export fab my_board.kicad_pcb -o ./gerbers --board-plot-params
openhac export fab my_board.kicad_pcb -o ./gerbers --ipc2581   # also IPC-2581 via kicad-cli
```

This writes Gerber layers, Excellon drill files (mm), and by default two CSV position files (front and back). **`--ipc2581`** adds an IPC-2581 export when your `kicad-cli` supports it (ignored by `.gitignore` pattern **`*.ipc2581`** if you generate into the repo tree).

### SPICE Simulation

Skip the PCB pipeline entirely and go straight to a SPICE netlist:

```python
from openhac.core import Board
from openhac.core.base import Module, Component
from skidl import Net

class RCFilter(Module):
    def __init__(self):
        super().__init__("RC_LowPass")
        self.r = self.add(Component("R_1k_0603"))
        self.c = self.add(Component("C_10uF_0805"))

        vin  = Net("VIN")
        vout = Net("VOUT")
        gnd  = Net("0")   # SPICE ground is always node 0

        self.r['1'] += vin
        self.r['2'] += vout
        self.c['1'] += vout
        self.c['2'] += gnd

        self.r.part.value = "1k"
        self.c.part.value = "10uF"

board = Board(size_mm=(10, 10))
board.add_module(RCFilter())
board.simulate("rc_filter")   # writes rc_filter.cir
```

### Searching the Component Database

```python
from openhac.database.db_manager import DatabaseManager

db = DatabaseManager()

# Exact lookup by generic name
part = db.get_component("R_10k_0805")
print(part["mpn"], part["supplier_sku"])   # RC0805FR-0710KL  C17513

# Parametric search
results = db.search_components(query="3.3V", category="voltage_regulators", limit=10)
for r in results:
    print(r["generic_name"], r["supplier_sku"])
```

### Adding a Custom Component

Any component not in the DB can be added manually:

```python
from openhac.database.db_manager import DatabaseManager

db = DatabaseManager()
db.insert_component({
    "generic_name":    "OLED_128x64_I2C",
    "kicad_symbol":    "Display:SSD1306",
    "kicad_footprint": "Display:OLED_128x64_I2C",
    "manufacturer":    "Solomon Systech",
    "mpn":             "SSD1306",
    "supplier_sku":    "C5443",
    "description":     "128x64 OLED display, I2C, 3.3V",
})
```

---

## Error Handling

The compiler raises structured exceptions (mostly from `openhac.core.base`; DRC from `openhac.compiler.rule_check`):

| Exception | When |
|---|---|
| `ERCFloatingNetError` | A net has fewer than 2 connected pins |
| `ERCUnconnectedPinError` | A pin has no net assignment |
| `ERCMissingPowerFlagError` | A power net has no PWR_FLAG |
| `ERCPowerBudgetError` | Total current draw exceeds supply |
| `UnconnectedInterfaceError` | A required module interface is not connected |
| `InterfaceNotFoundError` | `expose_interface()` called with unknown name |
| `FreeRoutingNotFoundError` | FreeRouting jar not found |
| `AutorouterFailedError` | FreeRouting exited with error or produced no SES |
| `FabExportError` | `kicad-cli` fabrication export failed or binary missing |
| `SchematicGenerationError` | SKiDL circuit unavailable at schematic generation time |
| `LayoutGenerationError` | KiCad `pcbnew` failed or is unavailable during layout |
| `DRCViolationError` | Design rule check failed (board bounds, IPC width, optional policies) |
| `RiskyPartLookupError` | Live/JIT part mapping blocked (low/medium confidence + strictness) |
| `KicadLibraryLoadError` | KiCad symbol could not load in strict KiCad mode |

---

## Project Structure

```
openhac/
  core/
    base.py       # Component, Module, Interface, all exceptions
    board.py      # Board — compiler entry point
  stdlib/
    mcu.py        # ESP32_WROOM
    power.py      # XT60_Input, LDO_5V
    passives.py   # Resistor, Capacitor
  compiler/
    rule_check.py     # ERC + DRC
    netlist_gen.py    # SKiDL → .net + BOM
    layout_gen.py     # Z3 → KiCad PCB outline + invokes footprint placement
    pcb_placement.py  # SKiDL parts → pcbnew footprints + NETINFO_ITEM
    autoroute_cli.py  # FreeRouting subprocess integration
    schematic_gen.py  # KiCad S-expression schematic
    spice_gen.py      # SPICE .cir netlist
    project_gen.py    # .kicad_pro project file
    export_fab.py     # kicad-cli Gerber/drill/pos export
    compile_manifest.py  # post-compile JSON manifest
  database/
    db_manager.py  # SQLite CRUD
    sync_jlc.py    # JLCPCB catalog sync via jlcsearch API
    seed_data.py   # Baseline hand-verified parts
    schema.sql     # DB schema
tests/             # pytest + Hypothesis property tests
scripts/example_build.py       # Minimal compile example (run after seed_data)
scripts/ci_full_compile_smoke.py  # Optional full-layout CI smoke (pcbnew)
```

### Git ignores

The root **`.gitignore`** excludes generated KiCad/SPICE artifacts, local **`openhac.db`**, pytest/Hypothesis/benchmark outputs, **`tests/tmp/`** / **`tests/fixtures/generated/`**, coverage files, **`*.ipc2581`**, **`.openhac/`**, KiCad **`*-backups/`**, and **`*-release.zip`**. **Keep `tests/**/*.py` and `tests/conftest.py` tracked** — they are required for CI (only caches and scratch under `tests/` are ignored).

---

## License

Open source. See LICENSE.
