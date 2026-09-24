# STAR.EXE — Original Binary & Reverse-Engineering Archive

This is the **frozen reverse-engineering debris** of a 16-bit MS-DOS Sri Lankan
Vedic astrology program (`STAR.EXE`), whose original author is unknown and whose
source code was never available. The program was freely distributed in Sri Lanka
and fell out of use only because modern operating systems can no longer run
16-bit DOS binaries.

This archive is **not built, not tested, and not re-linked** into the modern
port at `../STAR/` (or `git@github.com:madurapa/star.exe.git`). It exists for
provenance and arbitration only — a reference copy of the original binary and
the extractions that drove the port.

## What lives here

| Path | Contents |
|---|---|
| `STAR.EXE` | The 16-bit DOS binary. Run via [DOSBox-X](https://github.com/joncampbell123/dosbox-x): `dosbox-x STAR.EXE` |
| `STAR.EXE.i64`, `STAR.EXE.idc` | IDA Pro database artifacts |
| `STAR.EXE.asm` | Full disassembly (67,917 lines, base `1000h`, entry `1000:A46`) |
| `full_analysis_listing.asm` | Publics/symbols map only |
| `binary_blueprint.json` | Extracted string/layout blueprint (352 strings, 32 routines) |
| `STAR.EXE_export_for_ai/` | IDA Export for AI: one `.asm` per failed decompilation, `function_index.txt`, `strings.txt`, `imports.txt`, `exports.txt`, `pointers.txt`, `memory/` hexdumps |
| `scripts/` | One-shot extraction harnesses: `extract_all.py`, `dump_*.py`, `r2_stage*.py`, `expB.py`/`expC.py` (DOSBox drivers), `extract_strings.py`, `real48.py` (Real48 decoder) |
| `suggestion/` | Contemporary analysis notes and proposals |
| `r2_out/` | Unreferenced radare2 output |

The IDA Export (`STAR.EXE_export_for_ai/AGENTS.md`) documents 165 functions in
**legacy mode** (single file per function), with metadata headers recording
function name, address, callers/callees. Note: the IDA FLIRT analysis swapped
`@Sin`/`@Cos` names globally — this is a known critical porting detail, not
an extraction bug.

## What the reverse-engineering recovered

The original binary computes Sri Lankan Vedic horoscopes with these proven
routines (addresses in the disassembly):

- **`sub_2564E`** — Meeus Gregorian Julian Date: `JD = f(Y, M, D.frac)` with
  constants 100.0, 4.0, 2.0, 365.25, 30.6001, 1720994.5.
- **PROGRAM `0x1159D–0x118EA`** — IAU GMST block: `GMST0 = 24110.54541 +
  8640184.812866*T + 0.093104*T² - 0.0000062*T³`; LMST composes birth + UT
  + longitude. The GMST constant `24110.54541` differs from Meeus's `.54841`
  and is replicated as-is.
- **`sub_15FF8`** — fractional-year → Y/M/D splitter (Trunc/Trunc/Round +
  30-day month borrowing).
- **`sub_21EE1`** — H/M/S display splitter (Int/Int/Round + 59.99 carry
  threshold, banker's rounding).
- **`sub_220B3`** — planetary ephemeris engine (21 Sin, 25 Cos — read under
  the swap as cos/sin — 99 mul, 41 div, 2 Sqrt, 4 ArcTan).
- **`sub_18760`** — Lagna/ascendant engine (11 Sin, 10 Cos, 5 ArcTan).
- **`sub_23A58`** — nakshatra/pada finder + avastha dispatch:
  `idx = (DF4·DF6·E00 + E02 + DF8 + DFA) mod 12`.
- **`sub_1633A`** — Vimshottari dasa timeline (lord table Ravi 6 … Sikuru 20).
- **`sub_1D989`** — Kendraya ASCII chart renderer (GOTOXY + glyph writes).
- **`sub_1CBB2`** — string-concat chart builder.

The binary computes in **Borland Real48** (40-bit effective mantissa,
40-bit exponent bias 128). `dseg` is pure BSS (`?`); all tables live as inline
Pascal `ShortString` literals in the code segment, length-prefixed and labeled
by IDA. 400 strings were recovered by `extract_strings.py` /
`real48.py`. All 11-byte city/label tables, 27 YONI/RUXHA attribute tables,
yoga/karana/tithi/amsha names, and the 165-function disassembly are present in
the asm.

## Deliberately reproduced quirks (fidelity contract)

The modern port reproduces these binary-literal behaviors (documented in
`../STAR/docs/quirks.md`). They are not bugs in this port; they are the
contract — changing a number breaks the golden-file gate.

- **Non-carry seconds**: degrees split without carrying seconds into minutes
  (e.g. Shani `213:52:60`). Lagna never wraps down; negatives wrap up via
  `normUp`.
- **59.99 carry threshold**: time display promotes at ≥59.99, banker's-rounded.
- **JD rounds half-to-odd** (not half-even): glibc `printf` cannot be used.
- **30-day month borrowing** in dasa ages.
- **23|24 balance razor**: display shows balance `2-0-23` while the table age
  is `2-0-24`; the engine Moon sits 0.63" under the screen value.
- **City 13 mismatch**: slot labeled `MATHARA` but carries Batticaloa
  coordinates (7°40′/81°43′).
- **City 14**: labeled `BADULLA` but coordinates identify it as Batticaloa.
- Binary spellings kept verbatim: `NRAYANA` (Nirayana), `Chadra` (Moon,
  load-bearing for the hora remapping chain), `Rav1` (Ravi, digit-one typo),
  `Aakasha` (double-a), `Urenus`/`Neptun`, `Pluuto`, city labels
  `CLOMBO`, `A"PURA`, `K"GALA`, `POLONARU`.
- **Yoni field-width**: stored 12-char `ShortString`, displayed 10 chars.
- **Gana Keti nondeterminism**: Keti matches no Gana set; observed DOS outputs
  are city-dependent (Raxha ×6, Maanusha ×1, blank ×2). Port returns majority
  (Raxha).
- **Blank Linga for Siyavsa**: the `nak == 28` check is unreachable (probable
  typo for 24); Siyavsa yields a blank Linga.
- **Misplaced fixed karanas**: the dispatch table deviates from textbooks.

## Baseline checkpoints

Profile: **Test User**, 1981-12-08 12:55, Ratnapura (city 7), Nirayana.
Verified longitudes — Lagna 334:50:42, Chandra(Moon) 9:23:49, Ravi(Sun)
232:33:32, Budha 231:17:00, Sikuru 276:02:26, Kuja 152:42:30, Guru 188:32:33,
Shani 176:32:50, Raahu 90:52:13, Kethu 270:52:13. JD 2444946.809, Ayanamsa
23°34'21". Dasa opens Ketu Maha 2-0-23.

The modern port pins these with automated screen-diff harnesses
(`../STAR/tests/verifier.cpp`, 83+ assertions) and differential + boundary
fuzzing (106-case matrix).

## Relationship to the modern port

| Aspect | This archive | `../STAR/` (the port) |
|---|---|---|
| Role | Frozen extraction debris + original binary | Active C++ console app |
| Build status | Never compiled, never re-linked | Builds with CMake; `--verify` → `VERIFY_ALL_GREEN` |
| Ground truth source | Original DOS binary via DOSBox captures | Golden screen files (`tests/screens/`) |
| Extraction tooling | `scripts/*.py` (run once, kept for record) | `tools/check_extraction.py` only consumer (reads asm text) |
| Quirk handling | Binary-literal reference | Deliberately reproduced; documented in `docs/quirks.md` |

The re-baselining rule: golden files change **only** by explicit re-baselining
with recorded justification. Never edit expectations to match new code. If
disassembly is ever needed again, use free tooling (Ghidra/radare2) — the
proprietary tools that produced this archive have no place in the pipeline.

## License

The original binary is unlicensed freeware (abandonware). This archive is
preserved for reference. Provenance statement: see `../STAR/NOTICE.md`.
