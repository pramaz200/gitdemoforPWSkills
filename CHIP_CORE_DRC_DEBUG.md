# chip_core signoff debugging — XOR / Magic-DRC / KLayout-DRC / LVS

Observed on the institute PC (`runs/chip_core_recon`), all **deferred** (flow continued):

| step | result |
|---|---|
| `62-checker-xor` | **112** XOR differences |
| `65-checker-magicdrc` | **21466** Magic DRC errors |
| `66-checker-klayoutdrc` | **664** KLayout DRC errors |
| `70-checker-lvs` | **5** LVS errors |

> **Key fact you already found:** *raising die area / lowering placement density did not help.* That rules
> out chip-level congestion/spacing as the main cause and points at **intrinsic** sources:
> (1) **macro-internal DRC** — chip-level Magic DRC re-checks the *full merged GDS*, including everything
> inside `eFPGA_top` (which was **never DRC-signed-off** — we deferred it), and (2) **density fill**, which
> gets *worse* with more empty area. Fix those at the source, not the floorplan.

The single most important step is **§1 (spatially bin the errors)** — it tells you how much is inside the
macros vs. chip-level, which decides everything else.

---

## §0 — First, dump the actual rule names + locations (don't debug blind)

```bash
cd $PROJ
R=runs/chip_core_recon
# Magic DRC: error types + counts (Magic writes a .magic.drc / *.rpt with feedback)
find $R -path "*magic-drc*" -name "*.rpt" -o -path "*magic-drc*" -name "*.magic.drc*" 2>/dev/null
grep -iE "^\[|rule|spacing|width|density|enclosure|overlap|extension" $R/*magic-drc*/*.{rpt,drc,log} 2>/dev/null \
  | sed -E 's/[0-9]+\.[0-9]+/N/g' | sort | uniq -c | sort -rn | head -40
# KLayout DRC: rule (category) counts
grep -oE "<category>[^<]+</category>" $R/*klayout-drc*/*.lyrdb 2>/dev/null | sed -E 's#</?category>##g' \
  | sort | uniq -c | sort -rn | head -40
# LVS: the 5 mismatches
grep -iE "net|device|port|property|mismatch|unmatched|Final|Cell" $R/*netgen-lvs*/*.log $R/*lvs*/*.rpt 2>/dev/null | tail -60
# XOR: which layers/where
grep -oE "<category>[^<]+</category>" $R/*xor*/*.xml $R/*xor*/*.lyrdb 2>/dev/null | sort | uniq -c | sort -rn
```

---

## §1 — Spatially bin the Magic-DRC errors (THE deciding diagnostic)

`eFPGA_top` sits at `[1450,120]`–`[4567,7225]`; SRAMs at the lower-left (`100..1100 x 100..2300`).
Find how many of the 21466 errors fall **inside `eFPGA_top`** vs the SRAMs vs the rest (chip-level):

```bash
# Magic feedback coords are in the *.magic.drc / feedback; or convert the report to a layer and view.
# Quick numeric bin (extract x,y from the Magic DRC report, count inside the eFPGA bbox):
awk '/[0-9]+ +[0-9]+ +[0-9]+ +[0-9]+/{x=$1;y=$2;
  if (x>=1450000 && x<=4567000 && y>=120000 && y<=7225000) e++; else c++ }
  END{print "inside eFPGA_top:", e, " chip-level/other:", c}' \
  runs/chip_core_recon/*magic-drc*/*.rpt 2>/dev/null
# (adjust unit scale: Magic reports are usually in microns or DBU; check a few lines first)
```
Or visually in KLayout/Magic: load the DRC report as a marker layer over the GDS and see if the markers
cluster inside `eFPGA_top` (→ macro-internal) or along PDN/abutment edges (→ chip-level).

**Interpretation:**
- **Most errors inside `eFPGA_top`** → the fabric macro isn't DRC-clean → §2 (sign it off standalone). This
  is the expected outcome and explains the 21466 count + why floorplan changes don't help.
- **Most along PDN stripes / macro edges / everywhere (windows)** → density + PDN → §3, §4.

---

## §2 — Fix the bulk: DRC-sign-off `eFPGA_top` as a standalone macro FIRST

Hierarchical rule: **a macro must be DRC-clean before integration**, because chip-level Magic DRC re-checks
its internals. We deferred eFPGA_top's DRC, so its internal violations are landing in chip_core.

1. Run Magic DRC on the **eFPGA_top** run alone (on the ≥32 GB machine):
   ```bash
   librelane work/librelane/efpga_stitch/config.yaml --pdk gf180mcuD --pdk-root $PDK_ROOT --manual-pdk \
     --design-dir $PROJ --last-run --from Magic.WriteLEF --to Checker.MagicDRC \
     --skip Checker.SetupViolations --skip Checker.HoldViolations \
     --skip Checker.MaxSlewViolations --skip Checker.MaxCapViolations
   grep -c "^\[" runs/efpga_stitch/*magic-drc*/*.rpt
   ```
2. Categorize eFPGA_top's own DRC by rule (§0 commands on `runs/efpga_stitch`). Typical fabric offenders:
   - **Density** (poly/metal min density inside the tile array) → §3.
   - **Antenna** (we saw these) → already have the diode fix (`RUN_HEURISTIC_DIODE_INSERTION`); also re-check.
   - **Min-area / spacing on the switch-matrix routing** → fix in the tile hardening (the tiles were hardened
     with relaxed settings: `RUN_CTS:false`, etc.). If a *tile* type is dirty, re-harden that tile clean,
     re-stitch.
   - **Tile-abutment spacing** in the stitch (76 tiles with 160 µm gaps shouldn't abut-clash, but check the
     tile edges).
3. Iterate eFPGA_top to **Magic DRC = 0**, re-stitch, then re-integrate. Most of the 21466 should vanish.

> If a clean fabric DRC is impractical short-term, the alternative is **black-boxing** `eFPGA_top` at chip
> level: give chip_core only the macro's **LEF (abstract) + a DRC-clean abstract GDS** so chip-level DRC does
> not descend into it (sign the macro off separately). That is the standard SoC methodology.

---

## §3 — Density fill (a large share of the 21466, and why area made it WORSE)

gf180 enforces **minimum metal/poly/active density** per window. Empty area fails it — so **bigger die /
lower placement density INCREASES density violations**, exactly what you saw.

Mitigations (do all that apply):
1. **Confirm it's density:** in the §0 rule histogram look for rules like `*.density`, `MET*.DN`, `PL.DN`,
   poly/COMP density. If thousands are density → this is a major contributor.
2. **Insert fill** (the real fix): metal/poly density fill must be added before DRC.
   - LibreLane inserts **std-cell decap/fill** (`Odb.*Fill*`), but gf180 **metal density fill** usually needs
     a dedicated pass. Run the PDK's fill, e.g. KLayout fill, or Magic `cif` fill, or the OpenROAD
     `density_fill` (`density_fill -rules <pdk_fill_rules>`), then re-stream and re-DRC.
   - Verify the LibreLane fill steps actually ran for chip_core (they may be skipped if a checker upstream
     deferred); check `runs/chip_core_recon/*fill*`.
3. **Keep the die as small as routing/macros allow** (the opposite of what you tried) so there's less empty
   area to fill — but fill is still required.
4. For a research/MPW run, density is often **waived**; for a real tape-out it must be filled. Decide per
   your goal.

---

## §4 — Chip-level DRC (PDN, macro abutment, top routing) — the part floorplan DOES affect

For errors that bin **outside** the macros (along PDN stripes, at macro edges):
1. **PDN vias / enclosure / spacing:** check rules like `V*.*`, `MET*.*` near the stripes. Common when the
   SRAM Metal2/3 bridge or the macro power straps don't match grid. Verify `PDN_CFG` (the SRAM Metal2/3
   bridge) is applied and `PDN_MACRO_CONNECTIONS` names match the real instances.
2. **Macro abutment spacing:** ensure `eFPGA_top` and each SRAM have enough keep-out from each other and from
   the core ring; nudge `MACROS.instances` locations apart; raise the per-macro PDN halo.
3. **Off-grid:** macro `location`s must be on the manufacturing grid (multiples of the DBU/route pitch). Snap
   them (e.g., round to 0.005/0.01 µm or the track pitch). Off-grid macros cause many `offgrid` errors.
4. **Min-area / floating metal** on top routing → usually fixed by the router; if present, re-route or add
   fill.

---

## §5 — XOR: 112 Magic-GDS vs KLayout-GDS differences

XOR compares the two streamout tools' GDS. 112 is small. Steps:
1. **See which layers/where:** load both GDS in KLayout and the XOR markers; or §0 XOR category dump.
2. **Classify:**
   - On **label/text/property layers**, **fill**, or **via arrays** → almost always **benign** tool-rendering
     differences (Magic vs KLayout draw via cuts/fill slightly differently). Safe to waive for a research run.
   - On **drawn signal/PDN metal or implant/well** → **real**; one tool dropped geometry. Investigate that
     cell (often a macro GDS read difference).
3. **Reduce/avoid:** pick **one** streamout as the signoff GDS (KLayout is the usual signoff streamer) and run
   DRC/LVS on that one, instead of comparing two. In LibreLane you can rely on the KLayout GDS for signoff.
4. Re-check after §2/§3 (fixing macro/fill often removes XOR diffs too).

---

## §6 — LVS: the 5 errors

5 is small and fixable. Steps:
1. **Read the 5** (§0 LVS command). They’ll be one of: unmatched **net**, unmatched **device/instance**,
   **property** (e.g. transistor size), or a **port/pin** mismatch.
2. **Most likely here — power:** the eFPGA_top macro LEF we generated with **OpenROAD** (not Magic) + the
   deferred fabric power; or SRAM power. Check that:
   - `eFPGA_top` nl (the `.nl.v` you fed chip_core) matches its layout (use the **stitch's own** final
     `nl.v`, not a fill-stage netlist, and the **same** GDS).
   - `PDN_MACRO_CONNECTIONS` connect `VDD/VSS` for `eFPGA_top` and every `u_sram` (names must match the synth
     netlist exactly — read them from `runs/chip_core_recon/*/chip_core.nl.v`).
3. **Macro LVS black-box:** netgen should treat `eFPGA_top` and the SRAM as black boxes matched by port name
   (their internals are signed off separately). Ensure the macro `nl`/`lib` are declared so LVS abstracts
   them. A port-name mismatch between the macro LEF/GDS and the netlist causes exactly this kind of small
   LVS error.
4. Fix, re-run `Netgen.LVS` only (`--from Magic.SpiceExtraction --to Checker.LVS`), iterate to 0.

---

## §7 — Recommended order (most leverage first)

1. **§1 bin the Magic-DRC errors.** This decides everything.
2. If macro-internal dominates → **§2 sign off `eFPGA_top` DRC standalone, re-stitch, re-integrate.** (Biggest
   drop in the 21466.)
3. **§3 add density fill** (and stop enlarging the die).
4. **§4 chip-level PDN/abutment/off-grid** for the remainder.
5. **§6 LVS** (only 5 — quick once power/macro-ports are right).
6. **§5 XOR** last — likely waivable; confirm the diffs are on benign layers.

Re-run signoff after each step and watch the counts drop. Use `--from`/`--to` to re-run only the relevant
checker (e.g. `--from Magic.DRC --to Checker.MagicDRC`) so you don't repeat the whole flow.

## §8 — Why your two attempts didn't help (so you don't repeat them)
- **↑ die area** → more empty area → **more** density-fill violations, and doesn't touch macro-internal DRC.
- **↓ placement density** → same effect (spreads cells, more empty space) and doesn't touch macro internals.
- The fix is **at the source** (clean macros + fill), not the chip floorplan.
