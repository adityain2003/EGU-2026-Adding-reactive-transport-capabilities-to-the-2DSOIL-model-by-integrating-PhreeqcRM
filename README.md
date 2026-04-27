# EGU 2026 — Adding reactive-transport capabilities to the 2DSOIL model by integrating PhreeqcRM

Companion repository for the EGU 2026 conference presentation. It contains
self-contained codebases that benchmark a coupled **2DSOIL ↔ PhreeqcRM**
reactive-transport model against analytical solutions.

The repository is organised as a monorepo, with each top-level numbered
folder being one self-contained codebase / study.

```
.
├── 1_BENCHMARKING_STUDY/   ← codebase 1: analytical-solution benchmark (this drop)
└── (more codebases will be added here)
```

---

## 1_BENCHMARKING_STUDY — analytical-solution benchmark

Three exercises were performed against analytical references:

| # | Scenario                                 | Reaction terms                              |
|---|------------------------------------------|---------------------------------------------|
| 1 | Conservative transport (no reaction)     | none                                        |
| 2 | First-order decay of species A           | μ = 3 × 10⁻⁶ mol/sec                        |
| 3 | First-order decay + zero-order production| μ = 3 × 10⁻⁶ mol/sec, γ = 1 × 10⁻⁶ mol/sec  |

For Cases 2 and 3, μ (decay rate) and γ (zero-order production) are set in
the visualization / error-calculation Python scripts.

### Layout

```
1_BENCHMARKING_STUDY/
├── INFO.txt
├── 1_PHREEQC_BENCHMARKING_STUDY_1_ANALYTICAL_SOLUTION/   Python plotting + analytical reference
├── ERROR_CALCULATION/                                    Error-metric scripts (RMSE / MAE / SMAPE)
└── Maizsim_PhreeqcRM/
    ├── soil source/      2DSOIL (Fortran)  → builds 2dMAIZSIM.exe
    ├── crop source/      MAIZSIM (C++)     → builds Maizsim.dll
    ├── PHREEQCRM/        PhreeqcRM library source (vendored, USGS)
    ├── PHREEQCRM_BUILD/  CMake / MSBuild outputs for PhreeqcRM (.dll, .lib, databases)
    ├── TEST_CASE/        Sample scenario inputs
    └── maizsim07_PHREEQCRM.sln
```

### Components

- **2DSOIL** — Fortran finite-element soil-water/solute solver.
- **MAIZSIM** — C++ crop model, linked as a DLL.
- **PhreeqcRM** — USGS reactive-transport library (`PhreeqcRM.dll`).
- The Fortran ↔ C++ bridge to PhreeqcRM lives in
  `soil source/PhreeqcRM.FOR` and `soil source/RM_interface.F90`.

### Pre-built binaries

So the model can run on a target Windows x64 PC without rebuilding,
the following are checked in:

- `Maizsim_PhreeqcRM/soil source/x64/Debug/2dMAIZSIM.exe`
- `Maizsim_PhreeqcRM/soil source/x64/Debug/Maizsim.dll`
- `Maizsim_PhreeqcRM/soil source/x64/Debug/PhreeqcRM.dll`, `PhreeqcRMd.dll`
- `Maizsim_PhreeqcRM/PHREEQCRM_BUILD/{Debug,Release}/PhreeqcRM*.{dll,lib,exp}`
- PHREEQC chemistry databases (`*.dat`) under
  `Maizsim_PhreeqcRM/PHREEQCRM_BUILD/database/`

### Inputs and outputs (run-time files in `soil source/x64/Debug/`)

| File                          | Role                                                                 |
|-------------------------------|----------------------------------------------------------------------|
| `PHREEQCRM_FILENAME.txt`      | Input: name of the active `.pqi` chemistry runfile (Case 1 / 2 / 3). |
| `PHREEQCRM_RUNFILE_BENCHMARKING_CASE_{1,2,3}.pqi` | PhreeqcRM chemistry definitions per case.        |
| `FERTIGATION_SCHEDULE.txt`    | Input: dates / hours / solute-flux / water-flux for fertigation.     |
| `phreeqc.dat`, `Amm.dat`, `Amm_2DSOIL.dat` | PHREEQC thermodynamic databases.                        |
| `run.dat`, `runTEST_CASE_1.dat` | 2DSOIL run controllers.                                            |
| `TEST_CASE_BM/`, `TEST_CASE_BM_750CM/` | Soil and climate inputs for the benchmark grids.            |
| **`FERTIGATION_OUTPUT.txt`**  | **Main output** — concentrations of species A at every node where x = 0 (1-D profile), written at hour 0 of each day. |
| `PHREEQC_SPECIES_OUT.txt`     | PhreeqcRM species-concentration dump.                                |
| `PHREEQCRM_2DSOIL.chem.txt`, `PHREEQCRM_2DSOIL.log.txt` | PhreeqcRM-side chemistry / log output.      |

The `FERTIGATION_OUTPUT.txt` write is in
`soil source/FERTIGATION.FOR` (subroutine `FERTIGATION`, near line 124).
Columns: `NODE_NUM, TIME, DATE, HOUR, X, Y, CONC`.

### Run scenario

Per `INFO.txt`:

- **Days 1–5 (1–5 Jan 2024)** — media saturates, no solute applied.
- **Day 6 onwards** — solute is injected via the fertigation module; species A
  concentrations are written to `FERTIGATION_OUTPUT.txt` at hour 0 of each day.

### Visualization

Python scripts in `1_BENCHMARKING_STUDY/1_PHREEQC_BENCHMARKING_STUDY_1_ANALYTICAL_SOLUTION/`:

- `1_PHREEQC_2DSOIL_BENCHMARKING_SOLUTE_MOVEMENT.py` — Case 1 (no reaction).
- `2_PHREEQC_2DSOIL_BENCHMARKING_SOLUTE_BIODEGRADATION.py` — Cases 2 and 3
  (edit `mu` and `gamma` in the script to toggle between them).
- `3_PHREEQC_2DSOIL_BENCHMARKING_COMPARATIVE_CURVES.py` — comparative plots
  across all three cases.

Output figures (`Figure_1.png`, `PHREEQC_Benchmarking_Curves_*.{png,pdf,svg,eps}`)
are checked in alongside the scripts.

### Error metrics

`1_BENCHMARKING_STUDY/ERROR_CALCULATION/`:

- `ERROR_COMPUTATION_CASE_{1,2,3}_*.py` — per-case RMSE / MAE / SMAPE between
  simulated and analytical concentrations.
- `RESULTS_BENCHMARKING_ERROR_METRICS.{txt,xlsx}` — aggregated results.
- `COLUMN_SAMPLED_COORDINATES.xlsx` — sampling depths used for the metrics.

---

## Building from source

The Visual Studio solution is `1_BENCHMARKING_STUDY/Maizsim_PhreeqcRM/maizsim07_PHREEQCRM.sln`.
PhreeqcRM has its own build folder `PHREEQCRM_BUILD/` (CMake-generated VS solution).

Toolchain used:

- Visual Studio 2022 (`v143` platform toolset)
- Intel oneAPI Fortran Compiler (`ifx`) for the 2DSOIL Fortran project
- CMake 3.30 (used to (re)generate the PhreeqcRM build folder)

The Fortran linker is configured to pick up `maizsim.lib` from the C++
project's output and `PhreeqcRM.lib` from `PHREEQCRM_BUILD/Release/`
(see `soil source/2dMAIZSIM.vfproj`, configuration `Debug|x64`).

### Runtime requirements on the target PC

The shipped EXE is dynamically linked against:

- **Intel Fortran runtime** (`libifcoremd.dll`, `libifportmd.dll`, …) — install
  via the Intel oneAPI HPC Toolkit (or its redistributable runtime package).
- **Microsoft Visual C++ 2015–2022 Redistributable (x64)**.

Both are vendor components and are **not** redistributable through this repo.

---

## License

The vendored PhreeqcRM source under `Maizsim_PhreeqcRM/PHREEQCRM/phreeqcrm-master/`
retains its original USGS license (see that folder's own license / notice files).
The 2DSOIL and MAIZSIM components retain the licenses of their upstream
projects.
