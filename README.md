\# 3-Bay Industrial Dryer — Allen-Bradley Micro850 Control System



A ground-up PLC control system for a 3-bay industrial seed dryer, built on an Allen-Bradley Micro850 in Connected Components Workbench (CCW). The project runs in three phases — hardware/physical I/O, modular subsystem logic, and master sequencing/thermal safety — with each phase gated by its own commissioning and sign-off procedure before moving to the next.



\## Hardware



| Component | Spec |

|---|---|

| Controller | Micro850 (2080-L50E-24QWB) |

| Analog Input | 2080-IF4 (front plug-in, Module 3) — 4–20mA |

| Analog Output | 2080-OF4 (front plug-in, Modules 1 \& 2) — 0–10V / 4–20mA |

| HMI | PanelView 800, 2711R-T10T |

| Safety | Dual-channel E-Stop (pushbutton + pull-cable switch) through a hardwired industrial safety relay |



\*\*Tag naming convention:\*\* `PREFIX\_Subsystem\_Descriptor` — `DI\_`/`DO\_`/`AI\_`/`AO\_`/`VAR\_`/`CFG\_`. Enforced throughout; no loose naming like `Input\_1` or `Temp`.



\---



\## Phase 1 — Hardware Architecture \& Physical I/O Commissioning



Establishes the physical foundation: I/O tag database, wiring schematic, and a signed physical checkout sheet proving every input/output before any code runs.



\- \*\*Hardwired safety, not software safety.\*\* The safety relay's own dry power contacts cut the 24V DC / 120V AC output bus directly — the PLC is never in that path. `DI\_SafetyRelay\_Status` is a status \*input\* only; there is no PLC output that performs the safety cutoff.

\- \*\*Analog scaling.\*\* 4–20mA (temp, pressure) and 0–10V (fan speed) channels scaled via `SCALER` blocks, converting raw module counts to engineering units (32–200°F, 0–5" W.C., 0–60Hz).

\- \*\*Sign-off protocol:\*\* force outputs in CCW to confirm physical response (solenoids, VFD enables), actuate every field sensor and confirm bit state, verify E-Stop drops power in <50ms.



\---



\## Phase 2 — Modular Subsystem Development



Three reusable UDFBs (User-Defined Function Blocks), each instantiated rather than copy-pasted per physical device.



\### Bay Control Module

Instantiated three times (Bay1/Bay2/Bay3), each with private internal state. Inputs: `Bay\_Start\_Cmd`, `Run\_Time\_Setpt`, `Temp\_Actual`. Outputs: `Damper\_Solenoid\_Cmd`, `Basket\_VFD\_Run`, `Bay\_Active\_Flag`. Tracks runtime via a retentive timer; closes the damper and stops the VFD automatically on expiry.



\### Fan \& Air Balance Manager

Base speed staged linearly off active bay count (0/33/66/100% via `SCALER`), corrected by a live proportional pressure trim (±5Hz clamp) computed independently for Supply and Exhaust from their own pressure sensors and setpoints.



\### Burner Permissive Interlock Matrix

`Burner\_Enable\_Output` requires \*\*all four\*\*, every scan, with an instant drop on any single failure:

1\. `DI\_SafetyRelay\_Status` = TRUE

2\. Airflow proof confirmed (hardwired `DI\_SupplyFan\_Proof` / `DI\_ExhaustFan\_Proof` switches, sustained >10s via non-retentive `TON`)

3\. Purge cycle complete (End Damper open, fans run full purge duration)

4\. Ambient \& plenum temps below threshold



\---



\## Phase 3 — Master Sequence State Machine, Thermal Safety \& Commissioning



Integrates the Phase 2 UDFBs into one cohesive sequence via an integer-driven `System\_State`.



| State | Name | Summary |

|---|---|---|

| 0 | IDLE | All bay dampers closed, End Plenum Damper 100% open, fans/burner off |

| 10 | PRE-PURGE | End Damper open, fans at purge speed for a configurable duration |

| 20 | IGNITION PERMISSIVE | Purge complete + temp check + two-step HMI confirm before firing |

| 30 | ACTIVE RUNNING | Burner enabling permitted; Bay Control UDFB instances engaged |

| 40 | COOLDOWN | Burner disabled instantly, dampers reset, fans run cooldown profile |

| 99 | FAULT SHUTDOWN | Forced safe state; latched fault code; requires deliberate reset |



\*\*Three-tier Thermal Safety \& Alarm Cascade\*\* (independent of `System\_State`, runs in parallel across all states):



| Level | Threshold | Action | Recovery |

|---|---|---|---|

| 1 — Warning | ≥90°F | HMI banner + buzzer only | Auto-clears <85°F |

| 2 — Override | ≥95°F | Force End Damper open, bay dampers closed, inhibit new bay starts | Manual HMI reset once <90°F |

| 3 — Critical Trip | ≥105°F | Kill burner instantly, `System\_State` → 99 | Supervisor password reset + physical inspection |



\*\*Fault codes\*\* (`VAR\_FaultCode`, latched on entry to State 99): `1` = E-Stop, `2` = High-High Temp, `3` = Airflow Proof Lost.



\*\*Data logging:\*\* every 30s while in State 30 — Timestamp, `System\_State`, `Supply\_Temp\_F`, `Exhaust\_Press\_WC`, `Active\_Bay\_Mask`, `Burner\_Status`.



\---



\## Program Architecture (CCW)



Five Program files, fixed execution order (Micro850 programs run sequentially every scan — they cannot call each other):



1\. \*\*IO\_Scaling\*\* — raw counts → engineering units

2\. \*\*Safety\_Interlocks\*\* — permissive matrix, active bay count

3\. \*\*ManualControl\*\* — computes `Man\_\*` command values from jog/service-mode inputs, never writes physical outputs directly

4\. \*\*AutomaticControls\*\* — state machine reactions, Bay Control/Fan Balance UDFB calls, computes `Auto\_\*` command values

5\. \*\*Main\*\* — the single writer: arbitrates Manual/Auto/Off per `G\_System\_Mode` and is the only program that writes real `DO\_`/`AO\_` tags



Cross-program communication goes through Global Variables only; internal UDFB state stays Local/encapsulated. This "single-writer" discipline prevents two programs racing to drive the same physical output.



\---



\## Known Gaps \& Open Engineering Decisions



Deliberately tracked rather than silently assumed — several require sign-off from project leadership before being finalized:



\- \*\*Target pressure setpoints\*\* (`CFG\_Supply\_Pressure\_SP`, `CFG\_Exhaust\_Pressure\_SP`) and trim gain (`CFG\_Pressure\_Trim\_Gain`) are placeholder/test values — need real numbers from mechanical duct design or field commissioning, not from PLC logic itself.

\- \*\*End Plenum Damper position feedback\*\* does not physically exist yet. `AO\_EndDamper\_Position` is a commanded value only; a real `DI\_EndDamper\_Open\_Proof` limit switch is planned as a hardware/doc addition.

\- \*\*Thermal trip conflict resolved:\*\* the Phase 3 flowchart's single "95°F → Fault" arrow is superseded by the Alarm Cascade Matrix's three-tier structure (105°F is the actual State 99 trigger). Two additional hysteresis "clear" tags per tier are needed beyond the three `CFG\_Temp\_\*\_SP` tags currently defined.

\- \*\*`HMI\_Burner\_Ignite\_Confirm`\*\* and \*\*`CFG\_Cooldown\_Time\_SP`\*\* are referenced by Phase 3 requirements but not yet present in the Global Variables list.

\- \*\*Level 3 "supervisor password reset"\*\* has no defined mechanism — PanelView currently exposes only one flat `HMI\_Alarm\_Reset\_PB` with no privilege tiers.

\- \*\*Bay Control Module's retentive timer\*\* — the challenge spec names `TONR` (ControlLogix terminology); Micro850/CCW's actual equivalent is `RTO`, which does not auto-reset on falling edge and needs an explicit `RES` between bay cycles. Not yet fixed.

\- \*\*Mode selector (`G\_System\_Mode` / Hand-Off-Auto)\*\* is architecturally required but not present in the original PanelView tag table; intended as a physical 3-position selector, not an HMI-only control.

\- \*\*Bumpless transfer\*\* behavior when switching Manual → Auto mid-cycle is undecided.



\## Tooling



Task tracking in Linear — team \*\*Miller3Engineering\*\*, project \*\*FC Seed Processing\*\*. Issues organized under `Phase` and `Epic` label groups, one epic per Phase 3 requirement.

