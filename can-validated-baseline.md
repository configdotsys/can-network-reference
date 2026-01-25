# Project 4 - CAN Closure and Validated Baseline

*Author: Percival Segui*

*Prepared as an independent technical reference.*

*This document is a personal technical reference documenting validated operating behavior and baseline configuration from a completed CAN network project.*

---

## Purpose

Record the final, measured behavior of the CAN network and freeze a known‑good baseline for reuse.

This document is archival.

It captures *what was built*, *what was measured*, *what limits were observed*, and *what configuration is safe to reuse in future projects*.

---

### Scope

This document contains:

* final hardware configuration
* validated operating envelope
* measured stability limits
* observed failure thresholds
* recovery behavior
* recommended baseline settings
* improvement roadmap

This document does **not** re‑explain CAN theory, controller internals, or timing heuristics.

Those topics are documented elsewhere and are referenced only by outcome.

---

## Project Summary

**Objective:**

Establish a stable two‑node CAN network between heterogeneous STM32 microcontrollers and characterize its real‑world behavior under load.

**Nodes:**

* STM32F446RE
* STM32F767ZI

**Transceivers:**

* SN65HVD230 (3.3 V)

**Topology:**

* two‑node bench network
* twisted‑pair differential bus
* discrete termination at each end

---

## 1. Final Hardware Configuration

### 1.1 Electrical

* CANH / CANL: Cat‑5e twisted pair
* Ground: shared reference between nodes
* Termination: 120 Ω at each end (≈ 60 Ω measured)
* RS / Mode pin: tied to GND

### 1.2 Power

* each node powered independently via USB
* common ground provided through CAN loom

---

## 2. Clock and Timing Context

| MCU    | PCLK1  | TIMCLK |
| ------ | ------ | ------ |
| F446RE | 42 MHz | 84 MHz |
| F767ZI | 48 MHz | 96 MHz |

Nodes operated successfully with:

* different peripheral clocks
* matched baud‑rate intent

This confirmed that identical PCLK is not strictly required if timing math is consistent.

---

## 3. Final CAN Configuration

### 3.1 Nominal Baseline

* Bitrate: **500 kbps**
* Mode: Normal
* Auto retransmission: Enabled
* Auto bus‑off management: Enabled
* Filters: accept‑all

This configuration proved stable across extended run time.

---

### 3.2 Timing Characteristics

* sample point ≈ 80–85 %
* conservative segment allocation

Higher bitrates were explored but not selected as the final baseline.

---

## 4. Runtime Architecture

* interrupt‑driven RX
* software TX queue
* ring buffers for RX and error events
* UART telemetry in main loop only
* periodic heartbeat scheduling

This structure remained stable under continuous operation.

---

## 5. Measured Operating Envelope

### 5.1 Stable Region

| Parameter        | Stable     |
| ---------------- | ---------- |
| Bitrate          | ≤ 500 kbps |
| Application tick | ≤ 100 Hz   |
| Bus length       | ≈ 1 m      |
| Bus load         | < 10 %     |

Operation in this region produced:

* no sustained TEC growth
* no unintended bus‑off
* consistent RX/TX rates

---

### 5.2 Marginal Region

| Parameter    | Observation                |
| ------------ | -------------------------- |
| 500–750 kbps | occasional ACK errors      |
| ≥ 1 Mbps     | unstable with bench wiring |

Failures correlated strongly with:

* connector quality
* ground impedance
* cable movement

---

## 6. Observed Failure Behavior

### 6.1 ACK Loss

* dominant failure mode
* caused steady TEC increase
* preceded bus‑off

Root causes were electrical rather than firmware‑related.

---

### 6.2 Bus‑Off and Recovery

* disconnecting CAN lines reliably produced bus‑off
* ABOM restored operation automatically
* recovery time on the order of seconds

Firmware restart logic further improved robustness.

---

## 7. Diagnostic Instrumentation

The following telemetry proved critical:

* periodic heartbeat prints
* TX/RX rate counters
* TEC / REC monitoring
* decoded LEC reporting

These allowed immediate discrimination between:

* firmware stall
* bus silence
* electrical fault

---

## 8. Final Validated Baseline (Recommended Reuse)

The following configuration is recommended as a starting point for future CAN projects:

* Bitrate: **500 kbps**
* Sample point: ~80 %
* Accept‑all filters during bring‑up
* RX interrupt enabled
* ABOM enabled
* UART diagnostics active

This baseline emphasizes stability over throughput.

---

## 9. Lessons Confirmed

1. Physical integrity dominates CAN reliability.
2. Dupont‑style connectors are marginal at high speed.
3. Ground continuity is mandatory.
4. ACK loss is the primary bring‑up symptom.
5. Controller behavior is deterministic and predictable.
6. Diagnostics must exist from the beginning.
7. Conservative timing outperforms theoretical maximums.

---

## 10. Recommended Improvements (Next Iteration)

## Hardware

* replace loose jumpers with crimped or screw terminals
* add strain relief to prevent intermittent faults
* consider common‑mode choke on CANH/CANL

### Electrical

* prefer 500 kbps for bench networks
* verify termination before power‑up

### Firmware

* add TX‑empty interrupt support
* monitor queue watermark levels
* smooth diagnostic rate telemetry

---

## Closure Statement

This project successfully established a reliable two‑node CAN network and produced a reusable knowledge base covering:

* bring‑up
* controller behavior
  n- physical debugging
* timing heuristics
* validated operating limits

The documented baseline provides a stable foundation for future multi‑node and higher‑level CAN projects.

---

End of document.
