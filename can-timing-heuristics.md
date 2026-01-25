# CAN Timing and Configuration Heuristics

*Author: Percival Segui*

*Prepared as an independent technical reference.*

*This document is a personal technical reference for CAN bit timing and configuration heuristics used in embedded systems.*

---

## Purpose

Provide safe, reusable CAN bit‑timing patterns derived from industry practice.

This document bridges formal CAN timing theory with real‑world engineering heuristics used by OEMs and Tier‑1 suppliers.

It is intended to answer one question:
“What timing should I actually choose?”

---

### Scope

This document covers:

* CAN bit‑timing structure and terminology (concise recap)
* industry‑validated heuristic ranges
* sample‑point selection by bus length
* oscillator tolerance considerations
* canonical timing profiles for STM32 bxCAN
* portability method across different peripheral clocks

This document does **not** cover:

* CubeMX IOC setup
* CAN controller runtime behavior
* physical wiring and termination

Those topics are intentionally isolated elsewhere.

---

## Why Heuristics Exist

In production systems, engineers rarely derive CAN timing from propagation delay equations.

Instead:

* harness lengths are standardized
* transceiver delays are characterized
* oscillator tolerances are known
* EMC validation is expensive

Once a timing set passes validation, it becomes institutional knowledge.

> **Heuristics encode decades of measured bus behavior.**

---

## 1. CAN Bit Timing Structure (Recap)

One CAN bit consists of:

* SYNC_SEG = 1 TQ (fixed)
* PROP_SEG
* PHASE_SEG1
* PHASE_SEG2

Total bit time:

```
Tbit = (1 + TS1 + TS2) × TQ
```

Time quantum:

```
TQ = BRP / fPCLK
```

Resulting baud rate:

```
fbaud = fPCLK / (BRP × (1 + TS1 + TS2))
```

These identities are used to **verify** timing choices - not to invent them from scratch.

---

## 2. Industry Heuristic Ranges

### Typical validated ranges

| Bit rate  | Network length | Total TQ | Sample point |
| --------- | -------------- | -------- | ------------ |
| 1 Mbps    | ≤ 20 m         | 8 – 16   | 70 – 75 %    |
| 500 kbps  | ≤ 40 m         | 16 – 20  | 75 – 80 %    |
| 250 kbps  | ≤ 100 m        | 20 – 25  | 78 – 82 %    |
| 125 kbps  | ≤ 250 m        | 25 – 30  | 85 – 87.5 %  |
| 83.3 kbps | ≤ 500 m        | 30 – 40  | 87.5 – 90 %  |

These values appear consistently across:

* transceiver datasheets
* Vector / ETAS tools
* AUTOSAR timing templates

---

## 3. Practical Timing Rules

### Rule 1 - Choose total TQ first

* fewer than 8 TQ → poor granularity
* more than ~25 TQ → unnecessary jitter

Safe working range: **8–25 TQ**.

---

### Rule 2 - Sample point follows bus length

* short bus → earlier sample (≈ 70–75 %)
* long bus → later sample (≈ 80–87 %)

Later sampling allows more propagation margin.

---

### Rule 3 - Phase segments balanced

Typical ratio:

```
TS1 : TS2 ≈ 3 : 1
```

PHASE_SEG2 slightly shorter preserves resynchronization margin.

---

### Rule 4 - SJW selection

* SJW must be ≤ TS2
* common values: 1–2 TQ
* larger SJW tolerates oscillator drift

---

### Rule 5 - Sampling mode

* ≥ 500 kbps → single sampling
* ≤ 125 kbps → triple sampling (noise immunity)

---

## 4. Oscillator Tolerance Budget

ISO‑11898 requires that the combined oscillator error between any two nodes remains within limits.

Practical heuristic:

```
Total error < ±1.5 %
```

Typical crystals (±0.5 %) comfortably satisfy this.

Timing margin increases with:

* larger TS1
* reasonable SJW

Oscillator tolerance is part of timing design - not an afterthought.

---

## 5. Canonical Timing Profiles (STM32 bxCAN)

These profiles are widely reusable and intentionally standardized.

---

### Option A - Lean profile (Total TQ = 14)

Characteristics:

* sample ≈ 78.6 %
* good for short to medium buses
* excellent granularity

| Bit rate | BRP | TS1 | TS2 | Total TQ |
| -------- | --- | --- | --- | -------- |
| 1 Mbps   | 3   | 10  | 3   | 14       |
| 500 kbps | 6   | 10  | 3   | 14       |
| 250 kbps | 12  | 10  | 3   | 14       |
| 125 kbps | 24  | 10  | 3   | 14       |

This profile produces **exact baud rates** with APB1 = 42 MHz.

---

### Option B - Late‑sample profile (Total TQ = 21)

Characteristics:

* sample ≈ 81 %
* improved propagation margin
* one TS1/TS2 pattern across rates

| Bit rate  | BRP | TS1 | TS2 | Total TQ |
| --------- | --- | --- | --- | -------- |
| 1 Mbps    | 2   | 16  | 4   | 21       |
| 500 kbps  | 4   | 16  | 4   | 21       |
| 250 kbps  | 8   | 16  | 4   | 21       |
| 125 kbps  | 16  | 16  | 4   | 21       |
| 83.3 kbps | 24  | 16  | 4   | 21       |

This profile simplifies fleet‑wide configuration.

---

## 6. HAL Encoding Caveat (Critical)

STM32 HAL macros encode timing fields with offsets.

For example:

* `CAN_BS1_11TQ` may represent TS1 = 10 TQ
* SYNC segment is handled implicitly

**Always verify actual TS1/TS2 values in the HAL header file**:

```
stm32f4xx_hal_can.h
```

Never assume macro names map 1:1 to physical TQ counts.

---

## 7. Porting to Other Clocks

For a new MCU or clock:

1. Choose Option A or Option B
2. Fix Total TQ and TS1/TS2 ratio
3. Solve:

```
BRP = fPCLK / (fbaud × TotalTQ)
```

4. Ensure:

* TS1 ≤ 16
* TS2 ≤ 8
* Total TQ ≤ 25
* BRP ≤ controller limit

5. Verify on analyzer or scope

This method avoids ad‑hoc tuning.

---

## 8. When to Deviate

Deviate from heuristics only when:

* bus length exceeds assumptions
* oscillator quality is poor
* EMC testing indicates margin issues

In most bench and embedded projects:

> **heuristic timing is sufficient and safer than custom derivation.**

---

## Key Takeaways

* CAN timing is constraint‑driven
* sample point matters more than exact TQ count
* heuristics are validated, not arbitrary
* reuse timing profiles whenever possible
* always verify HAL encoding

---

### Document Status

This document defines **timing patterns**, not project‑specific settings.

It is intended for reuse across:

* future CAN projects
* different STM32 variants
* multi‑node systems

---

End of document.