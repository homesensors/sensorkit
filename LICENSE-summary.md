# License summary

The `homesensors` project is split across four repos, each licensed
appropriately for its content.

| Repo / content | License | SPDX identifier |
|---|---|---|
| **`homesensors/firmware`** (MCU code) | [MIT](https://opensource.org/licenses/MIT) | `MIT` |
| **`homesensors/hardware`** (KiCad schematic, PCB, BOM, Gerbers) | [CERN-OHL-P v2](https://ohwr.org/cern_ohl_p_v2.txt) | `CERN-OHL-P-2.0` |
| **`homesensors/homeassist`** (HA add-on, Python scripts) | [MIT](https://opensource.org/licenses/MIT) | `MIT` |
| **`homesensors/sensorkit`** (this repo — docs, protocol spec, architecture) | [CC-BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) | `CC-BY-SA-4.0` |

## What each license allows

### MIT (firmware + HA add-on)

- ✅ Commercial use
- ✅ Modification
- ✅ Distribution
- ✅ Private use
- ❌ Warranty
- 🟡 Requires the copyright notice to be preserved in copies

The most permissive of the well-known software licenses. You can take
the code, modify it, fork it into a commercial product, charge money
for it, and not contribute anything back — as long as the copyright
notice and license text travel with copies of the code.

### CERN-OHL-P-2.0 (hardware)

The permissive variant of the CERN Open Hardware Licence v2. Drafted
specifically for open hardware (schematics, PCB layouts, mechanical
designs).

- ✅ Commercial use (make and sell physical products from this design)
- ✅ Modification (fork the schematic / PCB)
- ✅ Distribution
- ✅ Private use
- ❌ Warranty
- 🟡 Requires the licence + notices to travel with the design files
- 🟡 If you sell a physical product, you must either provide the
     buyer with a copy of the source or tell them where to obtain it

Functionally the hardware analogue of MIT. The "P" stands for
permissive — there are also `CERN-OHL-W` (weakly reciprocal) and
`CERN-OHL-S` (strongly reciprocal) variants which require derivatives
to share alike. We chose `-P` for maximum reach; if you want to
preserve the open-source nature against re-closing, refer to `-S`.

### CC-BY-SA 4.0 (docs)

Creative Commons Attribution-ShareAlike 4.0 International.

- ✅ Share — copy and redistribute the material in any medium or format
- ✅ Adapt — remix, transform, and build upon
- ✅ Commercial use
- 🟡 Attribution — must credit the original
- 🟡 ShareAlike — derivative works must be licensed under the same license

The standard for open-content documentation (Wikipedia uses this).
Requires that documentation forks stay open, preventing someone from
republishing the docs commercially without sharing improvements.

## Why this split

- **Code is MIT.** Maximises adoption. Hobbyists, commercial integrators,
  and curious students can all use the code without legal anxiety.
- **Hardware is CERN-OHL-P.** Matches the MIT spirit — designed for
  hardware, broadly accepted in the open-hardware community, OSHWA-
  compatible.
- **Docs are CC-BY-SA.** Documentation is the "story" of the project;
  ShareAlike ensures forks of the story stay open, which protects the
  community-driven nature of the project's narrative.

## If you want to relicense

The maintainers are open to relicensing requests — e.g. dual-licensing
for a specific commercial use case. Open an issue on this repo with
your use case.
