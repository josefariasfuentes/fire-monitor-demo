# Fire Monitor Control Simulator

Unity demonstrator of an automated industrial fire monitor: fire detection, pan/tilt control, ballistic water discharge, fault diagnostics and independent emergency suppression. The screenshots and four recorded walkthroughs below document the current demonstration. This is a simplified engineering simulation, not a validated fire-protection design.

**Public demonstration only:** the Unity project and source code are maintained privately. This repository contains the project overview, screenshots and demonstration videos.

## Operator panel

**SIMULATION / LIVE STATE** shows supply pressure, valve command and actual state, flow, encoder angles, motor currents, limit contacts and the selected zone. Current is expressed as a multiple of nominal current (**×In**), not amperes; the two-decimal encoder readings expose simulated measurement noise.

![Simulation panel with live telemetry](docs/media/d00-simulation.png)

**EVENT LOG** offers **DETAILED** scan events and **SIMPLIFIED** milestones, newest first. A nominal cycle records detection/aim command → monitor reached the zone → actual water discharge → extinction/valve-close command. Both histories run while the tab is closed and are cleared by Restart. Reached confirms angular alignment, not water impact.

<details>
<summary>View the simplified event log</summary>

![Simplified log showing faults, recovery, aiming and discharge](docs/media/d00-logs.png)

</details>

## Recorded demonstrations

These descriptions follow the recorded operator actions. Durations are the complete MP4 durations; the diagnostic rules are described separately below.

### D01 — Normal suppression · 0:39

Ignite **A** and let the monitor aim, open the valve and extinguish it; repeat with **C**. The recording then shows LIVE STATE, followed by the detailed and simplified logs. Each discharge starts after alignment and ends when the fire clears.

<!-- PUBLIC_VIDEO_D01 -->
*Video upload pending.*

### D02 — Low pressure, emergency suppression and recovery · 1:34

Set supply pressure to **1 bar**: H001 is diagnosed before any fire. Ignite **A**; insufficient pressure blocks the monitor valve and emergency suppression activates after its delay. Once A is extinguished, stop emergency suppression manually.

Ignite **A and B**: the detection change rearms automatic emergency protection. While emergency suppression is still reducing both fires, raise pressure to **4 bar**. After pressure recovery, the monitor serves **A, then B**. Water and emergency suppression contribute together while both are active. Stop emergency suppression manually and review the simplified log.

<!-- PUBLIC_VIDEO_D02 -->
*Video upload pending.*

### D03 — Fire sensor, stalled pan motor and stuck valve · 1:25

Briefly inject **Sensor A failed**: the alarm appears, but removing it before the emergency delay expires prevents activation. Then inject **Pan motor stalled** and **Valve stuck closed**, and ignite **A**. The stationary pan draws **3 ×In**; confirmed overcurrent triggers emergency suppression. After extinction, stop emergency suppression manually.

Ignite **A and B** to rearm the emergency response, then remove the pan stall. The motor recovers and the monitor aims at A, but the jammed valve cannot open. Only after a permitted OPEN command remains unfulfilled does the **VALVE timeout** confirm that second fault. Remove the valve injection: water starts, A is extinguished, the valve closes for retargeting, and the monitor serves B. Finally, stop emergency suppression and show the simplified log.

<!-- PUBLIC_VIDEO_D03 -->
*Video upload pending.*

### D04 — Frozen pan encoder and motor power loss · 1:11

Inject **Pan encoder frozen** and ignite **A**. The mechanism moves with normal current while the encoder reading remains fixed; the current/encoder discrepancy timeout confirms the encoder fault and stops the affected axis. Emergency suppression follows. Remove the injection before extinction: recovery verification allows aiming and water discharge to resume. After A is out, stop emergency suppression manually.

Next, inject **Pan motor power lost** and ignite **C**. Movement is commanded, but current remains zero and the encoder does not progress. The **no-current diagnostic** confirms a motor fault and emergency suppression activates. Restore power: normal current confirms recovery, the monitor reaches C and discharges. Stop emergency suppression shortly before water completes extinction, then review the simplified log.

<!-- PUBLIC_VIDEO_D04 -->
*Video upload pending.*

## Diagnostic and control rules

The fixed **20 ms** scan processes operator commands, samples feedback, diagnoses faults, calculates control, commits outputs and advances the plant. Diagnosis uses observable feedback rather than fault-injection switches.

| Diagnostic | Confirmation in this model |
|---|---|
| **H001 — low supply pressure** | Pressure **<2 bar for 1 s**, including standby; clears at **≥2.2 bar**. |
| **PAN/TILT MOTOR — NoCurrent** | Applied movement demand with current **<0.1 ×In for 2 s**. Encoder inactivity is an observable consequence, not an additional condition of this alarm. |
| **PAN/TILT MOTOR — Overcurrent** | Applied movement demand with current **>2 ×In for 2 s**; the simulated jam draws **3 ×In**. |
| **PAN/TILT ENCODER — discrepancy timeout** | Movement demand and normal current **0.1–2 ×In**, without **0.2°** encoder progress for **2.5 s**. Progress resets the window; idle or abnormal current interrupts it. |
| **PAN/TILT ENCODER — mechanical limit** | An independent MIN/MAX contact is active and the encoder differs from its known angle by **>0.5°**. |
| **VALVE** | An OPEN command remains unfulfilled for **2 s**. A blocked OPEN command cannot diagnose a jammed valve. |
| **FIRE SENSOR** | An unhealthy fire sensor raises an alarm and requests emergency suppression even without a detected fire. |

Motor and encoder alarms survive extinction. Healthy current under drive confirms motor recovery. Removing an encoder injection acknowledges repair; a timeout fault permits a motion test while water remains blocked until current and encoder progress agree. A coherent limit reading clears a contact discrepancy. Removing the valve injection acknowledges its repair.

OPEN requires a valid ballistic solution, measured alignment, no blocking fault and no active limit contact. A fire-sensor fault alone permits service to another zone detected by a healthy sensor. Blocking faults command ideal immediate safety closure; actual flow still depends on valve state, pressure and area.

Emergency suppression activates after **3 s** for a fire-sensor fault, or detected fire plus a motor, encoder, valve or low-pressure fault. It stays active until manually stopped. Deactivation blocks the current detection state; a later detection change rearms protection. Target priority is **A > B > C**. Normal current does not, by itself, prove shaft movement outside this simplified model.

## Physical model and assumptions

| Variable | Demonstration setting |
|---|---|
| Supply / open valve area | **3 bar gauge upstream of valve / 2,000 mm²** initially; adjustable in the panel. D02 changes pressure to 1 and then 4 bar. |
| Water / nozzle | Density **1,000 kg/m³**; nozzle diameter **40 mm**; valve/nozzle discharge coefficients **0.70 / 0.95**. |
| Trajectory / impact | Low ballistic arc; gravity **9.81 m/s²** downward; maximum sampled arc length **50 m**; zone hit radius **1 m**; first intersected zone receives water. |
| Pan / tilt | **30 / 20°/s**; limits **−170…170° / −45…85°**; alignment tolerance **0.5°**; encoder noise **±0.03°**. |
| Motor / valve | Current **0 ×In** idle or unpowered, **1 ×In** under normal demand, **3 ×In** jammed; binary valve with **1 s** normal actuation. |
| Fire intensity | Normalized **0…1**; monitor suppression **0.20/s**, emergency **0.05/s**, combined **0.25/s**. A surviving fire regrows **0.10/s** without either agent. |

Steady incompressible flow through valve and nozzle restrictions in series, followed by a gravity-only trajectory:

```text
v = sqrt(2P / (ρ × [(An / (Cdv × Av))² + (1 / Cdn)²]))
Q = An × v                         r(t) = r0 + v0 × t + ½g × t²
```

Here P is pressure in Pa, ρ density in kg/m³, Av/An valve/nozzle areas in m², and Cdv/Cdn discharge coefficients. Pipe losses, pump transients, drag, wind, droplet breakup, motor electrical dynamics and thermal fire behavior are omitted. Water impact and emergency-agent effects have no travel delay; snowfall represents an independent room-wide agent. Jet width is exaggerated visually.

## Implementation and further work

Open with **Unity 6000.3.25f1 / URP 17.3.0**, load `Assets/Project/Scenes/FireMonitorDemo.unity` and enter Play, or run a Windows build with that scene first. Apply numeric fields with Enter or focus loss. **Restart** reloads the saved starting configuration and clears both logs; **×** closes the executable or exits Play in the Editor.

Next steps: calibrate current/contact diagnostics, model motor acceleration and supply dynamics, expand sensor/reachability supervision, and export persistent logs. Further checks include interrupted-water regrowth and operator overrides; the recordings demonstrate behavior rather than formal engineering validation.

Visual assets: Free Fire VFX URP and ANGRY MESH Snowman particles. Simulation code: `Assets/Project/Scripts`.
