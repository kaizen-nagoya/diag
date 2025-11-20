# diag
Cooperate in open source initiatives to streamline automotive diagnostic systems. Clarify what the diagnostic system's structure will be useful for, and focus on assembling that open source. Use generative AI to assemble materials as systematically as possible.

# Question
What is the best method to design Diagnostics system in vehicle, such as SW-C, CDD or ISRs in AUTOSAR.
# Answer by ChatGPT
Short answer up front: **Prefer AUTOSAR-native diagnostics (Dcm/Dem with SW-C runnables) for most diagnostic logic.** Use **CDD** only for hardware- or timing-critical code that cannot be expressed inside normal AUTOSAR SW-Cs/BSW APIs. **Keep ISRs tiny** — do minimal HW handling and defer processing to tasks / runnables. Follow ISO-26262 and separation-of-concerns for safety and maintainability.
Below is a practical guide and decision checklist so you can pick the best approach for each diagnostic function.
## 1 — Roles & where things belong (Classic AUTOSAR view)
* **SW-C (Application layer, RTE-exposed runnables)**
  * Place most diagnostic logic (interpreting faults, decision logic, UDS request handling callbacks) here.
  * Benefits: re-usable, testable, portable across ECUs, integrates with RTE and SW architecture, easier to unit test & trace for ISO-26262.
* **BSW: Dcm (Diagnostic Communication Manager) & Dem (Diagnostic Event Manager)**
  * Use **Dcm** for implementing UDS services and communication; **Dem** for event/fault storage, counters, freeze-frames, status indicators.
  * These are standardized AUTOSAR modules — prefer them for compliance (UDS/OBD semantics) and tool support.
* **CDD (Complex Device Driver)**
  * Use when you must access non-AUTOSAR hardware, do DMA / linelike special HW handling, or need extreme timing determinism that SW-C/BSW APIs can't meet.
  * Keep CDD small, well-documented, and provide an AUTOSAR wrapper/API to the upper layers.
* **ISRs**
  * Do *only* the absolute minimum: acknowledge interrupt, capture timestamps or data into a lock-free buffer, set an event/flag, wake a task or call a short OS hook.
  * Never put heavy diagnostic logic or UDS processing the ISR.
## 2 — Decision checklist (choose SW-C vs CDD vs ISR)
Use this quick decision table:

| Criteria                                             | Use SW-C (Runnables) |                  Use CDD |              ISR |
| ---------------------------------------------------- | -------------------: | -----------------------: | ---------------: |
| Needs AUTOSAR services/UDS/DEM integration           |                    ✅ |                        ❌ |                ❌ |
| Portability / reuse / unit testing                   |                    ✅ |               ⚠️ limited |                ❌ |
| Hard realtime (< few µs response) or DMA/HW blocks   |             ⚠️ maybe |                        ✅ | ✅ (only capture) |
| Access to special HW registers / complex HW protocol |       ⚠️ via wrapper |                        ✅ |                ❌ |
| Memory/safety-critical low-level drivers             |                   ⚠️ | ✅ (but follow ISO-26262) |                ❌ |
| Simplicity & maintainability                         |                    ✅ |                       ⚠️ |     ✅ (but tiny) |

Guideline: prefer SW-C + Dcm/Dem. Pick CDD only when hardware or deterministic timing forces it. ISRs only for immediate HW handling and deferral.
## 3 — Architecture pattern (recommended)
Textual diagram:

Sensors / HW → **Device Driver / CDD** (low-level HW handling, DMA)
  ⇓ (safe API)
RTE → **SW-C: Diagnostic Manager / Diagnostic Application** (runnables)
  ⇓
BSW: **Dem** (store events), **Dcm** (UDS services), COM stack (PduR, Com) → CAN/LIN/Ethernet
User tool / Service tool ↔ Dcm over transport
Key rules:
* Expose CDD functionality to SW-C through well-defined AUTOSAR interfaces (so upper layers remain portable).
* Let Dcm handle UDS decoding where possible; only implement custom UDS semantics if necessary.
* Dem should be the single source of truth for event status and storage.
## 4 — Real-time and ISR handling pattern (pseudo)
ISR should only sample and notify:
```c
// pseudo-ISR
void ADC_IRQHandler(void) {
  uint16_t sample = ADC->DR;
  ring_buffer_push(&adcSamples, sample);   // lock-free or IRQ-protected
  SetEvent(DiagnosticTask, EVENT_ADC_SAMPLED); // wake deferred processing
}
```
Deferred runnable (SW-C / task):
```c
// Diagnostic runnable
if (Event_ADC_SAMPLED) {
  while (ring_buffer_pop(&adcSamples, &s)) {
    if (isOutOfRange(s)) {
      Dem_ReportErrorStatus(DIAG_ID_SENSOR_OUT, DEM_EVENT_STATUS_FAILED);
      // or notify SW-C logic via RTE
    }
  }
}
```
## 5 — Integration with AUTOSAR diagnostics (practical tips)
* **Use Dcm** for standard UDS (e.g., ReadDTCInformation, ClearDiagnosticInformation, TesterPresent). Avoid re-implementing UDS unless you must.
* **Use Dem** for event storage, aging, thresholds, and freeze frames. Keep diagnostic events consistent between SW-C and Dem.
* **Map diagnostic events** to signals and signals to NV storage or NVRAM via Dem configuration. Use tooling (RTE generator, configuration tools) to keep traceability.
* **State management**: use Mode Manager or RTE modes to handle ignition/diagnostic session modes; be explicit about which actions are allowed in which mode.
## 6 — Safety, testing, and lifecycle
* Follow **ISO-26262**: traceability from requirements → code → tests; perform failure mode and effects analysis (FMEA) on diagnostic reactions.
* Test levels: unit tests (SW-C), integration (RTE + BSW), SIL/HIL for ECU-level diagnostic behavior with a tester tool.
* Use logs and freeze frames to debug intermittent faults; design diagnostics to collect minimal but sufficient context.
* Plan for firmware update: ensure diagnostic data layout/versioning and migration strategies.
## 7 — Common anti-patterns (what to avoid)
* Putting UDS parsing or heavy logic inside ISRs.
* Large monolithic CDDs that bypass Dem/Dcm and scatter diagnostic event handling across driver code.
* Duplicated fault bookkeeping (two places tracking same DTC). Dem should be authoritative.
* Tightly coupling diagnostic logic with application UI or non-diagnostic concerns.
## 8 — Short recipe to implement a new diagnostic feature
1. **Write requirement**: what DTC, which freeze frame, severity, reaction, tester commands.
2. **Decide placement**: If needs HW direct: CDD + tiny ISR; otherwise SW-C + RTE + use Dem/Dcm.
3. **Define interfaces**: RTE ports, Dem event IDs, Dcm services.
4. **Implement ISR (if needed)**: minimal, push to buffer and set event.
5. **Implement SW-C runnable**: perform checks, call `Dem_ReportErrorStatus()` or RTE provided APIs.
6. **Configure Dcm/Dem** in your AUTOSAR tooling, map event IDs to DTCs and NVRAM.
7. **Unit + integration tests**; then HIL test with diagnostic tester (UDS tool).
## Final recommendation (one line)
Design diagnostics as **AUTOSAR-native**: use **SW-Cs + Dcm/Dem** for correct, testable, maintainable diagnostics; rely on **CDD** only for unavoidable low-level/timing hardware access; keep **ISRs** minimal and defer processing to tasks/runnables.
If you want, I can:
* sketch a concrete AUTOSAR module mapping for a specific diagnostic use case (e.g., ABS sensor fault handling),
* or produce example RTE interface and Dem/Dcm configuration snippets — tell me which ECU/diagnostic use case and I’ll draft it.
# requext
sketch a concrete AUTOSAR module mapping for a specific diagnostic use case (e.g., ABS sensor fault handling),
# Answer by ChatGPT
## AUTOSAR Module Mapping — ABS Wheel Speed Sensor Fault Handling
**Purpose:** sketch a concrete AUTOSAR mapping for detecting and handling an ABS wheel-speed sensor fault (e.g., open circuit, short to Vbat/GND, implausible signal).

## 1. High-level architecture
Sensors / HW
→ **Device Driver (ADC/GPIO/EXTI / CDD if needed)**
→ **RTE / SW-C: AbsDiag (Diagnostic SW-C)**
→ **BSW: Dem (Diagnostic Event Manager), Dcm (Diagnostic Communication Manager)**
→ COM / PduR / CanIf → CAN bus → Service Tool / Gateway
Notes:
* Low-level hardware sampling and any timing-critical signal decoding are implemented in a *CDD* (Complex Device Driver) or well-tested Device Driver module.
* ISR is used only to capture timestamps or to place raw samples into a lock-free buffer and notify an OS task.
* ABS application logic (traction control / braking logic) remains separate from diagnostics; diagnostics only observe and report.
## 2. Modules and responsibilities
| Layer       | AUTOSAR Module / Component          | Responsibility                                                                        |
| ----------- | --------------------------- | ------------------------------------------------------------------------------------------------- |
| HW access   | CDD / Device Driver (WheelSpeedDrv) | Configure ADC/timer/GPIO, sample sensor, use DMA, provide raw sample API (`GetRawWheelSpeedSample()`). Raise lightweight IRQ for samples.                                                     |
| ISR         | ISR handler (in CDD)                | Quick capture of sample or timestamp and push to ring buffer; set event for DiagnosticTask.                          |
| RTE         | SW-C: AbsDiag                       | Periodic runnable(s) to process samples, validate signal quality, detect faults, apply filter/thresholds, call `Dem_ReportErrorStatus()` when needed, expose diagnostic status via RTE ports. |
| BSW         | Dem                                 | Store DTCs, aging, freeze frames, event status; provide API used by AbsDiag.                              |
| BSW         | Dcm                                 | Expose UDS services (ReadDTCInformation, ClearDiagnosticInformation, ReadDataByIdentifier for freeze-frame).        |
| Upper apps  | SW-C: ABS_Control                   | Uses wheel speed signals for control; subscribes to status from AbsDiag through RTE (Mode/Status port).              |
| Persistence | NvM / NVRAM Manager                 | Store persistent DTC data or counters via Dem configuration.                                           |
## 3. Interfaces (RTE / Ports / APIs)
**AbsDiag SW-C ports**
* **Provided:** `AbsDiagStatus` (Mode/Status runnable output) — e.g., `enum {OK, DEGRADED, FAILED}` per wheel
* **Required:** `WheelSpeedRawApi` (client-server) — `GetSample(wheelId) -> sample struct` (from CDD wrapper)
* **Require/Callouts:** `Dem_ReportErrorStatus(eventId, status)` (BSW call)
**CDD / WheelSpeedDrv API**
```c
typedef struct { uint32_t timestamp_us; uint16_t rawValue; bool valid; } WheelSample_t;
Std_ReturnType GetRawWheelSpeedSample(uint8 wheelId, WheelSample_t* out);
```
**Dem event mapping**
* Event IDs (Dem): `DEM_EVT_WHEEL_FL_OPEN`, `DEM_EVT_WHEEL_FL_SHORT_GND`, `DEM_EVT_WHEEL_FL_IMPLAUSIBLE` (repeat for FR, RL, RR)
* Map each event to a DTC (e.g., `0xC12345` family) and freeze-frame sets.
## 4. Runnable design (AbsDiag SW-C)
**Runnables**
* `Rte_AbsDiag_SampleProcessing` (periodic, e.g., 5 ms)
  * Read X latest samples via `WheelSpeedRawApi` for each wheel
  * Compute frequency, edge timing or pulse width if Hall/variable reluctance
  * Validate range, signal amplitude, noise (SNR), and plausibility against vehicle speed (from CAN/VehicleSpeed SW-C)
  * Update internal fault counters / state machine
  * On confirmed fault transition: call `Dem_ReportErrorStatus(eventId, DEM_EVENT_STATUS_FAILED)`
  * On fault cleared: call `Dem_ReportErrorStatus(eventId, DEM_EVENT_STATUS_PASSED)`
* `Rte_AbsDiag_PeriodicMaintenance` (e.g., 1s)
  * Handle aging, counters, and prepare freeze frames if new event reported.
**State machine example (per wheel)**
* `OK` → if `signal_lost` for N consecutive checks → `FAULT_PENDING` → `FAULT_CONFIRMED` (report to Dem)
* `FAULT_CONFIRMED` → if `signal_good` for M consecutive checks → `FAULT_CLEARED` (report pass to Dem)
## 5. ISR / CDD pseudo-code
```c
// minimal ISR inside CDD
void EXTI_WheelSensor_IRQHandler(uint8 wheelId) {
  uint32_t t = Timer_GetMicros();
  WheelSample_t s = { .timestamp_us = t, .rawValue = GPIO_ReadCapture(wheelId), .valid = true };
  ring_buffer_push(&wheelBuffers[wheelId], &s);
  SetEvent(AbsDiagTask, EVENT_WHEEL_SAMPLE);
}
```
CDD exposes a non-blocking API; AbsDiag uses `GetRawWheelSpeedSample()` to pop from buffer.
## 6. DEM / DCM configuration notes
* **Dem**
  * Define EventParameter for each wheel fault with qualifier (failed, pending, pre-fail if OBD-relevant).
  * Configure freeze-frame data set (include: raw sample timestamps, vehicle speed, ignition status, last N samples). Keep freeze-frame size limited.
  * Configure storage class and NVRAM mapping via NvM for persistent DTCs and failure counters.
* **Dcm**
  * Map UDS services to Dem as usual. Ensure that `ReadDataByIdentifier` returns diagnostic status data exposed by AbsDiag if needed.
## 7. Safety & testing checklist
* Trace each diagnostic requirement to AbsDiag runnables and Dem event IDs (for ISO-26262).
* Unit tests for AbsDiag algorithm (fault detection thresholds, debounce counters).
* Integration tests: CDD + ISR + AbsDiag + Dem on SIL/HIL; inject jitter, noise, short/open faults.
* HIL test script: simulate open circuit, short to GND/Vbat, speed mismatch (vehicle speed vs wheel speed), verify DTC set/cleared and freeze-frame contents.
* Fault injection for intermittent faults: verify aging & debouncing prevents false positives.
## 8. Example DTC / Freeze frame fields
* **DTC:** `0xC1 23 45` (example)
* **Freeze frame fields:** ignition state, vehicle speed, wheelId, last 8 sample timestamps & values, sensor supply voltage, temperature (if available)
## 9. Deployment / versioning tips
* Keep CDD minimal and stable; version-driver ABI and document RTE port contract.
* Keep diagnostic thresholds configurable via calibration (NVRAM or ECU-config) so you can tune in the field without code changes.
## 10. Short sequence flow (Event: wheel open circuit)
1. ISR stops receiving pulses; CDD detects `no-pulse` for a sample window.
2. AbsDiag runnable observes missing pulses for `N` cycles → move to `FAULT_PENDING`.
3. After `M` pending cycles → `FAULT_CONFIRMED`; AbsDiag calls `Dem_ReportErrorStatus(DEM_EVT_WHEEL_FL_OPEN, FAILED)`.
4. Dem stores DTC, increments counters, records freeze-frame via configured dataset.
5. Dcm returns DTC on Tester `ReadDTCInformation` requests.
If you want, I can also:
* provide an example AbsDiag runnable implementation in C (RTE stubs), or
* produce a PlantUML diagram of the architecture, or
* produce a small test vector HIL plan (fault injection timeline).
Tell me which one and I'll add it.
Done — I created the detailed AUTOSAR ABS sensor diagnostic mapping in the canvas titled **"AUTOSAR ABS Sensor Diagnostic Mapping"**.
Open it in the document panel to view the full sketch.
Want the C runnable code for `Rte_AbsDiag_SampleProcessing` next, or a PlantUML architecture diagram?

