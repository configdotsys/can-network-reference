# CAN Bring-up and STM32 IOC Reference

## Purpose:
*Fast, repeatable path to a working CAN bus on STM32.*

*This document is procedural. It captures what to configure and why those settings matter during bring-up.*

*Conceptual theory, controller internals, physical-layer diagnostics, and timing heuristics intentionally live in separate documents.*

---

### Scope

This reference covers:

* CubeMX / IOC configuration required to bring CAN online
* Clock-tree constraints relevant to bxCAN
* Pin mapping and alternate-function selection
* Minimal feature set for reliable bring-up
* UART telemetry configuration for visibility
* Transceiver mode requirements

It does **not** cover:

* CAN protocol theory
* bxCAN internal state machines
* Error-counter interpretation
* Physical-layer debugging methodology
* Timing heuristics beyond what is required to make the bus operational

---

### Supported Targets

* STM32F4 family using **bxCAN**
* Validated on:

  * Nucleo-F411RE
  * Nucleo-F446RE

(Other STM32 families may require clock or pin adaptations.)

---

## Bring-up Philosophy

During initial CAN bring-up:

* **Visibility beats optimization**
* **Simplicity beats feature completeness**
* **Determinism beats throughput**

This document therefore prioritizes:

* identical timing across nodes where possible
* accept-all filtering
* minimal interrupts
* explicit clock control

Advanced features (filters, bus-off tuning, performance optimization) are intentionally deferred.

---

## 1. System Configuration

### 1.1 Toolchain

* HAL drivers (default CubeMX selection)
* No RTOS
* No middleware

Rationale:

* bxCAN behavior is easier to observe without scheduler interaction
* ISR timing remains deterministic

---

### 1.2 Debug Interface

* **SYS → Debug:** Serial Wire (SWD)

Required to:

* preserve debugger access
* avoid pin conflicts

---

## 2. Clock Configuration

### 2.1 Core Principle

> **CAN bit timing is derived from APB1 (PCLK1), not SYSCLK.**

Therefore:

* SYSCLK may differ between MCUs
* APB1 must be known and controlled

---

### 2.2 Recommended Bring-up Strategy

For early bring-up:

* force **APB1 = 42 MHz** on all nodes
* reuse identical CAN timing parameters

This minimizes variables during first communication tests.

---

### 2.3 Example Clock Trees

#### STM32F411RE

* SYSCLK = 84 MHz
* AHB = /1 → 84 MHz
* **APB1 = /2 → 42 MHz**
* APB2 = /1 → 84 MHz

#### STM32F446RE

* SYSCLK = 168 MHz
* AHB = /1 → 168 MHz
* **APB1 = /4 → 42 MHz**
* APB2 = /2 → 84 MHz

> HSI or HSE may be used. The critical requirement is APB1 = 42 MHz.

---

## 3. GPIO and Pin Mapping

### 3.1 CAN Pins

#### F411RE

* PA11 → CAN1_RX (AF9)
* PA12 → CAN1_TX (AF9)

#### F446RE

* PB8 → CAN1_RX (AF9)
* PB9 → CAN1_TX (AF9)

Notes:

* Pin locations differ by MCU variant
* Alternate-function selection must be verified explicitly

---

### 3.2 UART Telemetry Pins

Both boards:

* PA2 → USART2_TX (AF7)
* PA3 → USART2_RX (AF7)

UART is mandatory during bring-up.

---

### 3.3 Pin Conflict Check

Before code generation:

* verify CAN pins do not overlap ADC, SPI, or timer channels
* preserve prior-project peripherals if this project builds incrementally

---

## 4. USART Configuration (Telemetry)

### Required Settings

* Mode: Asynchronous
* Baud rate: 115200
* Word length: 8 bits
* Parity: None
* Stop bits: 1
* Flow control: None

Rationale:

* deterministic logging
* compatible with all terminal tools

DMA is not required.

---

## 5. CAN Peripheral Configuration

### 5.1 Operating Mode

Set:

* Mode: Normal
* Time-triggered mode: Disabled
* Auto retransmission: Enabled
* Receive FIFO locked: Disabled
* Transmit FIFO priority: Enabled

Disable initially:

* Auto bus-off
* Auto wake-up

(These may be enabled after baseline communication is confirmed.)

---

### 5.2 Bit Timing (Bring-up Baseline)

Assuming:

* APB1 = 42 MHz
* Target bitrate = 500 kbps

Configuration:

* Prescaler = 6
* Total time quanta = 14

  * BS1 = 11 TQ
  * BS2 = 2 TQ
  * SJW = 1 TQ

Result:

* 42 MHz / (6 × 14) = 500 kbps
* Sample point ≈ 85.7%

This configuration is conservative and bring-up friendly.

---

### 5.3 Filters

Initial configuration:

* Filter mode: Mask
* Filter scale: 32-bit
* Filter ID: 0x00000000
* Filter mask: 0x00000000
* FIFO assignment: FIFO0
* Filter activation: Enabled

Effect:

* accepts all CAN frames
* maximizes observability

Filtering should be introduced only after stable communication is achieved.

---

### 5.4 Interrupts

Enable:

* CAN1 RX0 interrupt

Optional (later):

* CAN TX interrupt
* CAN SCE (status/error) interrupt

RX0 interrupt is sufficient for bring-up and receive validation.

---

## 6. Transceiver Requirements

Validated with:

* SN65HVD230 (3.3 V logic)

Critical requirement:

* **RS / Mode pin must be tied to GND**

If RS is left floating or pulled high:

* transceiver enters standby or slope-control mode
* ACK loss and apparent firmware failure may occur

---

## 7. Minimal Bring-up Checklist

Before flashing firmware:

* [ ] APB1 confirmed at 42 MHz
* [ ] CAN pins mapped correctly (AF verified)
* [ ] RS pin tied to GND
* [ ] UART output functional
* [ ] CAN filter accept-all active
* [ ] RX0 interrupt enabled

After flashing:

* [ ] UART banner prints
* [ ] CAN peripheral starts successfully
* [ ] RX interrupt fires when bus traffic present

---

## 8. Bring-up Completion Criteria

CAN bring-up is considered successful when:

* frames transmit without mailbox saturation
* frames are acknowledged by another node
* RX interrupts fire reliably
* no continuous TEC growth is observed

Once these are met:

* enable advanced diagnostics
* enable auto bus-off management
* introduce filtering

---

### Document Status

This document is intentionally narrow in scope.

It defines a **known-good starting point**.

All deeper understanding, optimization, and debugging belongs in the companion documents:

* CAN Controller Internals and Runtime Behavior
* CAN Physical Layer and Debugging
* CAN Timing and Configuration Heuristics
* Project 4 Closure and Validated Baseline

---

End of document.
