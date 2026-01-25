# CAN Physical Layer and Debugging

*Author: Percival Segui*

*Prepared as an independent technical reference.*

*This document is a personal technical reference for CAN physical-layer behavior, wiring, and debugging in embedded systems.*

---

## Purpose

Identify, diagnose, and resolve CAN failures that originate outside firmware.

This document captures the *physical realities* of CAN networks - wiring, grounding, termination, connectors, and failure signatures - and maps them to observable runtime symptoms.

Most persistent CAN problems are electrical or mechanical, not software.

---

### Scope

This document covers:

* CAN physical-layer fundamentals as they appear in practice
* termination and impedance requirements
* grounding strategy
* transceiver configuration
* wiring and connector quality
* bench-top failure modes
* symptom → indicator → cause mapping
* practical debugging workflow

This document does **not** cover:

* CubeMX configuration
* CAN controller internals
* CAN timing heuristics

Those topics are intentionally isolated in other references.

---

## Core Principle

> **If the bus is electrically unhealthy, correct firmware will still fail.**

CAN controllers strictly enforce protocol rules.
When the physical layer violates assumptions, the controller responds with:

* ACK errors
* TEC growth
  n- Error Passive transitions
* Bus-Off

Firmware often appears "broken" even when it is behaving correctly.

---

## 1. Differential Signaling Reality

CAN uses differential signaling:

* **Recessive bit:** CANH ≈ CANL ≈ 2.5 V
* **Dominant bit:** CANH ≈ 3.5 V, CANL ≈ 1.5 V

Key implications:

* noise couples equally onto both lines and cancels
* common-mode voltage must remain within transceiver limits

Loss of ground reference shifts common-mode voltage and breaks communication.

---

## 2. Termination Requirements

### 2.1 Standard Termination

* 120 Ω resistor at each physical end of the bus
* ≈ 60 Ω total measured across CANH–CANL (power off)

Anything else will cause reflections.

---

### 2.2 Verification Procedure

With power removed:

1. Measure resistance between CANH and CANL
2. Expected value ≈ 60 Ω

Results:

* ≈ 120 Ω → one terminator missing
* ≫ 120 Ω → open circuit
* ≪ 60 Ω → excessive termination

This measurement alone eliminates many failure modes.

---

## 3. Grounding Strategy

CAN requires:

* shared signal reference
* low-impedance return path

Even though signaling is differential, **ground is not optional**.

Best practice:

* dedicate a twisted pair for CANH/CANL
* provide a parallel ground conductor
* tie grounds at both nodes

Symptoms of poor ground:

* intermittent ACK errors
* TEC spikes when cables move
* erratic bus-off recovery

---

## 4. Transceiver Configuration

### 4.1 RS / Mode Pin (SN65HVD230)

For SN65HVD230-class transceivers:

* RS pin must be tied to GND

If RS is:

* floating
* pulled high

Then the transceiver may enter:

* standby mode
* slope-control mode

Resulting symptoms:

* transmitter appears functional
* receiver never ACKs
* firmware appears faulty

---

## 5. Wiring and Connectors

### 5.1 Twisted Pair Requirement

CANH and CANL must be tightly twisted:

* Cat 5e twisted pair works well
* loose parallel wires do not

Twisting ensures equal noise coupling.

---

### 5.2 Stub Length

Untwisted stubs act as antennas.

Recommended:

* < 1 cm exposed lead at each node

Long stubs introduce reflections and phase distortion.

---

### 5.3 Connector Quality

Common bench failures originate from:

* Dupont jumper wires
* loose crimps
* cold solder joints
* unsupported fine-gauge strands

Observed behavior:

* moving the cable causes ACK failures
* TEC increases when wires are touched

CAN reliability is mechanical as much as electrical.

---

## 6. Physical Failure Signatures

### 6.1 ACK Error Dominance

The most common physical failure symptom:

* ACK errors reported by transmitter
* rising TEC

Typical root causes:

* missing ground
* transceiver standby
* broken CANL
* incorrect termination

The controller cannot distinguish these causes.

---

### 6.2 Bus-Off Trigger Patterns

Bus-Off typically appears when:

* transmitting continuously
* no valid ACKs occur

Observed behavior:

* mailboxes fill
* TEC reaches 256
* controller disconnects from bus

This is a *protective response*, not a malfunction.

---

## 7. Debugging Workflow (Recommended)

Follow this order strictly:

1. **Measure termination** (≈ 60 Ω)
2. **Verify ground continuity**
3. **Confirm RS pin tied to GND**
4. **Inspect connectors and solder joints**
5. **Reduce bitrate to 500 kbps or lower**
6. **Use accept-all filters**
7. **Observe TEC and LEC counters**

Changing firmware before completing steps 1–4 often wastes time.

---

## 8. Observable Symptoms and Likely Causes

| Symptom                          | Likely Physical Cause      |
| -------------------------------- | -------------------------- |
| No RX frames                     | Broken CANH/CANL           |
| ACK errors                       | Ground or transceiver mode |
| Works when touching wires        | Mechanical joint issue     |
| Works at 100 kbps but not 1 Mbps | Reflection / impedance     |
| Intermittent bus-off             | Connector movement         |
| Stable after rewiring            | Confirmed physical fault   |

These patterns repeat consistently across projects.

---

## 9. Practical Bench Lessons

Validated observations:

* Dupont jumpers are unreliable above ~500 kbps
* Cat 5e twisted pair significantly improves stability
* Parallel ground conductor reduces common-mode noise
* Strain relief prevents intermittent faults

Physical robustness directly increases protocol reliability.

---

## 10. Key Takeaways

* Most CAN failures originate outside firmware
* ACK loss is the dominant symptom
* Termination and grounding are mandatory
* Mechanical integrity matters
* Measure before modifying code

When CAN behaves erratically, assume physics first.

---

### Document Status

This document serves as a **field-debugging reference**.

It is intended to be consulted when:

* CAN works intermittently
* errors appear non-deterministic
* firmware changes have no effect

Timing heuristics and controller internals are intentionally documented elsewhere.

---

End of document.
