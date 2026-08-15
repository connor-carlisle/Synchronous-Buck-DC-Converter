# 24V → 5V Synchronous Buck Converter

A 3A, 500 kHz synchronous buck DC-DC converter designed from scratch in Altium Designer around the TI LM5145 controller. Built as a self-directed learning project to understand switching power supply design end-to-end — from operating-point math through compensation, PCB layout, and design-for-assembly — with an emphasis on being able to justify every design decision rather than chase maximum optimization.

**Designer:** Connor Carlisle · **Date:** 2026-08-15 · **Rev:** A · **Tools:** Altium Designer

---

## Specifications

| Parameter | Value |
|---|---|
| Input voltage | 24 V |
| Output voltage | 5 V |
| Max output current | 3 A |
| Switching frequency | 500 kHz |
| Control method | Voltage-mode with input feedforward |
| Compensation | Type III |
| Predicted efficiency | ~82% at full load |
| PCB | 4-layer, 1 oz copper |

---

## Why these choices

The point of this project was to make defensible decisions, not optimal ones. The reasoning behind the major ones:

**Voltage-mode control with Type III compensation.** The LC output filter contributes a double pole that dumps ~180° of phase lag into the loop — in voltage mode this would oscillate. A Type III compensator injects two zeros to add phase back around crossover. The network was validated with TI's LM5145 Quickstart Calculator to ~55 kHz crossover and ~65° phase margin: fast, and comfortably stable.

**Synchronous rectification.** The low-side MOSFET replaces the freewheeling diode, trading a diode's ~0.5 V forward drop for a few milliohms of Rds(on) — the core reason synchronous topologies are efficient. Dead-time between the FETs prevents shoot-through.

**Efficiency as a known tradeoff, not a failure.** ~82% at full load, with the dominant loss being high-side MOSFET switching loss. This is understood and defensible — a deliberate consequence of the switching frequency and part selection — rather than something the design tries to hide.

**Single ground plane over a split AGND/PGND.** The design originally implemented a separate analog ground joined to power ground at a single net-tie. During layout I evaluated the tradeoff and consolidated to a single PGND plane while keeping the feedback and compensation components tightly clustered near the IC and away from the switch node. At 3 A / 500 kHz, a well-placed single plane captures most of the noise benefit without the routing fragility of a star ground — the right tradeoff for this design.

---

## Architecture

**Schematic (hierarchical):**
- `Top.SchDoc` — block diagram
- `Input_Protection.SchDoc` — input connector, polyfuse, reverse-polarity Schottky, TVS clamp, power LED
- `Power_Stage.SchDoc` — half-bridge FETs, inductor, hybrid input/output capacitor banks, snubber footprints (DNP), output connector
- `Controller.SchDoc` — LM5145, Type III compensation, feedback divider, soft-start, current limit, bootstrap, PGOOD, debug header, test points

**PCB stackup (4-layer):**
- Top — components + power routing
- L2 — solid ground plane (return path directly beneath the hot loop)
- L3 — power (VIN / VOUT)
- Bottom — routing

---

## Design highlights

- **Minimized hot loop.** The high-di/dt commutation loop (input HF ceramic → high-side FET → low-side FET) is placed as a tight loop with the ground return in the plane directly beneath, minimizing parasitic inductance and switch-node overshoot.
- **Thermal management.** Direct-connect copper and via arrays under the FET, inductor, and controller thermal pads move heat into the ground plane; the high-side FET (the hottest part) gets the most via real estate.
- **Design for assembly.** Wettable-flank controller package chosen for inspectable solder joints; board laid out for solder-paste stencil + reflow given the QFN/SON thermal pads that can't be hand-soldered. Snubber footprints populated on the board (DNP) so damping can be tuned during bring-up without a respin.
- **Bring-up readiness.** Four test points (VIN, VOUT, SW, GND) and a debug header for probing during first power-on.

---

## Key components

| Function | Part |
|---|---|
| Controller | TI LM5145RGYR (VQFN-20) |
| High-side FET | TI CSD18563Q5A (NexFET SON 5×6) |
| Low-side FET | TI CSD18531Q5A (NexFET SON 5×6) |
| Inductor | Würth 7447797100, 10 µH |
| Output bulk cap | Panasonic 16SVPG47M (polymer) + Murata X7R ceramics |
| Reverse-polarity protection | SS5P10 Schottky |
| Transient protection | SMBJ30A TVS |
| Input protection | Resettable polyfuse |

---

## Engineering process

The most valuable part of this project was the debugging. Several real errors were caught during PCB import and design-rule verification by cross-referencing pad net assignments back to the schematic — a loop I applied throughout:

- **High-side FET drain-source inversion.** Q1's drain was wired to PGND instead of VIN, inverting the high-side switch. Caught by reading the pad net on the PCB against the intended topology.
- **Missing decoupling grounds.** The high-frequency input cap and both VCC decoupling caps had their ground terminals unconnected — leaving them electrically inert. Found the same way and corrected on the schematic.
- **VCC-to-PGND short.** The controller's VCC rail was accidentally tied to PGND, shorting the internal regulator. Traced and cut.

Each fix was made on the schematic and re-imported, never patched on the PCB — because net assignments live in the schematic and hand-editing the layout desyncs the two. These would have been genuinely hard to debug post-fabrication.

---

## Status

- [x] Specification and operating-point analysis
- [x] Component selection and BOM
- [x] Schematic capture (hierarchical)
- [x] Compensation design and validation
- [x] PCB layout, routing, and pours
- [x] Design-rule verification clean
- [ ] Fabrication and assembly
- [ ] Bring-up and bench validation

---

## What I learned

- Layout is not a formality on a switching supply — the hot loop, ground return, and decoupling placement determine whether the board is quiet or a mess, and the penalty for getting it wrong is far sharper than on lower-frequency designs.
- The distinction between "electrically correct" and "well-designed": a board can pass connectivity checks and still perform poorly if the loop is loose or the analog placement is careless.
- Package selection drives assembly method — bottom-pad SON/QFN parts require reflow, and designing for the assembly process you actually have is its own discipline.
- Systematic debugging (reading nets back to intent) catches errors that visual inspection misses.

---

## Repository structure

```
/hardware        Altium project (schematic + PCB)
/docs            Design notes, calculations, datasheets
/fabrication     Gerbers, drill files, BOM, pick-and-place (pending)
/images          Renders and layout screenshots
README.md
```
