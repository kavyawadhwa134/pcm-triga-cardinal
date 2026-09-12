# pcM — Handoff to Codex

Send this file as-is. Fill in the block below first; everything else is ready.

---

## ⚠️ FILL THIS IN BEFORE SENDING

```text
CODE REPO:        <github url for this repo>
BINDER REPO:      <github url of the Binder image repo>
BINDER LAUNCH:    <mybinder.org launch link>

Cardinal is ALREADY INSTALLED AND WORKING in that Binder image.
Use it. Do not rebuild Cardinal from source.
Image definition lives in:  <Dockerfile | environment.yml | postBuild | apt.txt>

NUCLEAR DATA:     ENDF/B-VII.1 is what the Binder image has.
                  THE APPROVED SPEC REQUIRES ENDF/B-VIII.0. See the warning
                  immediately below - this is a decision, not a detail.
                  OPENMC_CROSS_SECTIONS=<path>/cross_sections.xml

MEASURED LIMITS:  32 GiB RAM, <nproc> cores, MPI = <OpenMPI | MPICH>
                  -> Tier 3 (Full). NekRS polynomialOrder 7 fits (10.07 GB).

YOUR TASK:        <state exactly what Codex should accomplish>
```

## 🛑 STOP — nuclear data library conflict

`spec/materials.md` and every result in this repo use **ENDF/B-VIII.0**. The
target environment has **ENDF/B-VII.1**. These are different libraries.

This matters because pcL — the team pcM is being compared against — used
ENDF/B-VIII.0. Running pcM on VII.1 means any later pcM-vs-pcL difference is a
**mix of code difference and library difference, with no way to separate them.**
That destroys the comparison this whole exercise exists to make.

The evaluations that changed between VII.1 and VIII.0 include thermal
scattering and the U isotopes — precisely what drives a ZrH-moderated TRIGA.
A shift of several hundred pcm is plausible, which is at or above the
supervisor's own stated ">500 pcm means something differs" threshold. In other
words, **running VII.1 could by itself trigger a false "something differs"
finding.**

CLAUDE.md lists "nuclear data unavailable" and "two approved sources conflict"
as explicit STOP conditions, and forbids silently changing nuclear data.

**Do not just run VII.1 and report the number.** Choose one:

1. **Preferred** — put ENDF/B-VIII.0 HDF5 in the image (subsetting to the
   nuclides this model uses is fine: U-235, U-238, Zr isotopes, H-1, O-16,
   Fe/Cr/Ni/Mn/Si/C, plus `c_H_in_ZrH`, `c_Zr_in_ZrH`, `c_H_in_H2O`).
2. **Ask the supervisor** whether VII.1 is acceptable for this comparison.
3. **If VII.1 is used anyway** — label every result `ENDF/B-VII.1` prominently,
   state that it is NOT the spec library, and do not present the numbers as
   comparable to the VIII.0 results already in `results/`.

A cheap way to size the effect: run the unit cell on both libraries and report
the k-infinity difference. That converts an unknown into a measured number.

If you don't have the four measured values, run this inside a live Binder
session and paste the output:

```bash
which cardinal-opt; echo $OPENMC_CROSS_SECTIONS; nproc; free -g
mpirun --version | head -1; mpifort -show
```

---

## What this is

**pcM** — an independent TRIGA Mark I reactor simulation on Cardinal
(OpenMC neutronics + MOOSE heat conduction + NekRS CFD).

**Read `docs/RUNBOOK.md` in the repo before running anything.** It is
self-contained and is the authority on everything summarized here.

## The rule that overrides everything

A separate team (**pcL**) is independently simulating the **same reactor** in
GeN-Foam. A supervisor compares the two afterward. The entire value of the
exercise depends on neither side seeing the other's work.

- **Never** read, copy, or consult anything from pcL.
- **Never** tune a pcM parameter so results agree with pcL.
- **Never** compute or state pcL-vs-pcM agreement — that is the supervisor's job.
- If the only way past a blocker is looking at pcL: **stop and report instead.**

Also: never fabricate a number, never silently change geometry / nuclear data /
energy groups, and never call solver convergence "validation."

## First commands

```bash
git clone <CODE REPO> pcm && cd pcm
./scripts/audit_environment.sh
```

The audit auto-detects Cardinal, Python, nuclear data, NekRS, the Fortran
toolchain and any GPU on macOS or Linux, then prints `OK`/`MISSING` per
component plus a NekRS memory-sizing table for the machine it's on.

If it misdetects, override **before** it runs and re-run it:

```bash
export PCM_CARDINAL_DIR=<dir containing cardinal-opt>
export PCM_CROSS_SECTIONS=<path>/cross_sections.xml
export PCM_CORES=<n>
```

Resolve every `MISSING` your tier needs. Nothing else in the repo hardcodes a
path.

## Running

```bash
./scripts/run_openmc.sh unitcell baseline      # standalone neutronics
./scripts/run_coupled.sh smoke                 # OpenMC <-> MOOSE, ~2 min
./scripts/run_coupled.sh production 8          # 10 Picard steps, 8 ranks
./scripts/run_nekrs.sh 8                       # standalone fluid, 8 ranks
```

Work in that order. Don't start an expensive run before its smoke test.

## Verify you haven't broken the physics

```bash
$PCM_PYTHON cardinal/openmc/temperature_coefficient.py --config baseline
```

Expect **≈ −6.34 pcm/K** average over 296–1000 K, becoming *more* negative as
fuel heats. That is the hydrogen-in-ZrH signature and it is what distinguishes
a TRIGA from a generic pincell.

**A small or positive coefficient means the `S(α,β)` data is missing or wrong.
Stop and fix that before trusting any other result.**

## Four traps that will cost you hours

Full detail in `RUNBOOK.md` Part 6.

1. **Never launch NekRS on one MPI rank.** Nek5000 uses compile-time static
   allocation; one rank sets `lelt` = the whole mesh and demands ~4.25 GB in a
   single compilation unit. This killed the development machine — `gfortran`
   took SIGTERM. `run_nekrs.sh` refuses `<2` ranks; **leave that guard in.**
   More ranks fix the *compile*, not total memory — if total RAM binds, shrink
   the mesh or lower `polynomialOrder`.
2. **`mpifort` must actually work, not just exist.** On conda MOOSE installs it
   wraps a toolchain compiler in the env's `bin`; if that isn't on `PATH` the
   NekRS build dies ~70 files in with `command not found`.
3. **`c_H_in_ZrH` thermal scattering data is mandatory.** Without it results
   look plausible but the defining TRIGA physics is gone.
4. **Cardinal's `lwr_solid` / `pincell_multiphysics` tutorials are syntax
   references only.** Their physics is wrong here: TRIGA fuel radius 1.815 cm
   vs ~0.39, pitch 4.235 vs ~1.26, U-ZrH/SS304 vs UO₂/zircaloy, 293 K pool vs
   553 K pressurized. This must return nothing:
   `grep -rniE '0\.39218|0\.45720|3000e6|zircaloy|uo2' cardinal/`

## Container-specific

Cardinal is already installed — that part is solved. What still bites:

- **Measure real limits from inside**, don't assume: `nproc; free -g; df -h /tmp`
- **Session timeouts kill long runs.** Run detached, log to a file, and write
  results to persistent storage — not container-local `/tmp`.
- **MPI under containers often needs flags** — e.g. `--allow-run-as-root` on
  OpenMPI, or forcing a shm/tcp transport.

## State at handoff

**Done and verified:**

| | |
|---|---|
| k-infinity, 2-D unit cell | 1.384207 ± 0.000214 (21 pcm) |
| k-effective, 3-D bare core | 1.102059 ± 0.000260 (26 pcm) |
| Coupled OpenMC↔MOOSE | max fuel 358.8 K, max clad 336.0 K |
| Power conservation | exact (2083.33 W) |
| Analytic cross-check | within 1.1% |
| Fuel temperature coefficient | −6.34 pcm/K |
| NekRS mesh pipeline | complete, `fluid.re2` committed |

**NOT done — do not report these as done:**

- Coolant temperature, velocity, pressure — **NekRS has never run**
- Converged coupled production run (the smoke result carries ~2% tally error)
- Three-way OpenMC ↔ MOOSE ↔ NekRS coupling
- MGXS generation (Phase 6)
- The +0.5 $ reactivity transient (deliberately out of baseline scope)

**Two caveats to carry forward:**

1. Results are for the **average pin**. Pin peak/average is 2.2544, so a hot
   channel is materially hotter.
2. The convective `h = 1098 W/m²·K` accounts for **65%** of the temperature rise
   while being applied at Re = 4755 — transitional, outside Dittus–Boelter's
   reliable range. It is the weakest link in the thermal result and the main
   reason to run NekRS. See `spec/thermal.md`.

## Reporting

Update `PCM_STATUS.md` at milestones. Final numbers go in
`results/PCM_RESULTS.md` with software versions, hardware, nuclear data,
particle counts, mesh, convergence, runtime, exact reproduction commands, and
an explicit limitations section.

For every number, state which verification level it has reached — software /
numerical / physics / cross-code / experimental. Do not collapse them into
"validated."

**Report blockers rather than working around them.** Substituting a different
code, library, or geometry to produce *a* number is not an acceptable outcome.
Saying "this environment cannot support it" is.
