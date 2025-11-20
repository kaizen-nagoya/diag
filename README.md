# diag
Cooperate in open source initiatives to streamline automotive diagnostic systems. Clarify what the diagnostic system's structure will be useful for, and focus on assembling that open source. Use generative AI to assemble materials as systematically as possible.

## Question
What is the best method to design Diagnostics system in vehicle, such as SW-C, CDD or ISRs in AUTOSAR.
## Answer by ChatGPT
Short answer up front: **Prefer AUTOSAR-native diagnostics (Dcm/Dem with SW-C runnables) for most diagnostic logic.** Use **CDD** only for hardware- or timing-critical code that cannot be expressed inside normal AUTOSAR SW-Cs/BSW APIs. **Keep ISRs tiny** — do minimal HW handling and defer processing to tasks / runnables. Follow ISO-26262 and separation-of-concerns for safety and maintainability.
Below is a practical guide and decision checklist so you can pick the best approach for each diagnostic function.

# 1 — Roles & where things belong (Classic AUTOSAR view)
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
# 2 — Decision checklist (choose SW-C vs CDD vs ISR)
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

# 3 — Architecture pattern (recommended)
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
# 4 — Real-time and ISR handling pattern (pseudo)
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
# 5 — Integration with AUTOSAR diagnostics (practical tips)
* **Use Dcm** for standard UDS (e.g., ReadDTCInformation, ClearDiagnosticInformation, TesterPresent). Avoid re-implementing UDS unless you must.
* **Use Dem** for event storage, aging, thresholds, and freeze frames. Keep diagnostic events consistent between SW-C and Dem.
* **Map diagnostic events** to signals and signals to NV storage or NVRAM via Dem configuration. Use tooling (RTE generator, configuration tools) to keep traceability.
* **State management**: use Mode Manager or RTE modes to handle ignition/diagnostic session modes; be explicit about which actions are allowed in which mode.
# 6 — Safety, testing, and lifecycle
* Follow **ISO-26262**: traceability from requirements → code → tests; perform failure mode and effects analysis (FMEA) on diagnostic reactions.
* Test levels: unit tests (SW-C), integration (RTE + BSW), SIL/HIL for ECU-level diagnostic behavior with a tester tool.
* Use logs and freeze frames to debug intermittent faults; design diagnostics to collect minimal but sufficient context.
* Plan for firmware update: ensure diagnostic data layout/versioning and migration strategies.
# 7 — Common anti-patterns (what to avoid)
* Putting UDS parsing or heavy logic inside ISRs.
* Large monolithic CDDs that bypass Dem/Dcm and scatter diagnostic event handling across driver code.
* Duplicated fault bookkeeping (two places tracking same DTC). Dem should be authoritative.
* Tightly coupling diagnostic logic with application UI or non-diagnostic concerns.
# 8 — Short recipe to implement a new diagnostic feature
1. **Write requirement**: what DTC, which freeze frame, severity, reaction, tester commands.
2. **Decide placement**: If needs HW direct: CDD + tiny ISR; otherwise SW-C + RTE + use Dem/Dcm.
3. **Define interfaces**: RTE ports, Dem event IDs, Dcm services.
4. **Implement ISR (if needed)**: minimal, push to buffer and set event.
5. **Implement SW-C runnable**: perform checks, call `Dem_ReportErrorStatus()` or RTE provided APIs.
6. **Configure Dcm/Dem** in your AUTOSAR tooling, map event IDs to DTCs and NVRAM.
7. **Unit + integration tests**; then HIL test with diagnostic tester (UDS tool).
# Final recommendation (one line)
Design diagnostics as **AUTOSAR-native**: use **SW-Cs + Dcm/Dem** for correct, testable, maintainable diagnostics; rely on **CDD** only for unavoidable low-level/timing hardware access; keep **ISRs** minimal and defer processing to tasks/runnables.

If you want, I can:

* sketch a concrete AUTOSAR module mapping for a specific diagnostic use case (e.g., ABS sensor fault handling),
* or produce example RTE interface and Dem/Dcm configuration snippets — tell me which ECU/diagnostic use case and I’ll draft it.
