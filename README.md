# Edge-AI Co-Processor for Intelligent VFD Systems


## Summary

### Problem

Variable Frequency Drives (VFDs) rely on deterministic control loops (e.g. PWM, field-oriented control) but lack the ability to interpret high-frequency motor signals such as phase currents, voltage ripple, and vibration patterns.

Existing systems use threshold-based fault detection, which:
- reacts late to developing faults
- cannot capture subtle time-series patterns
- leads to unplanned downtime and reduced equipment lifespan

Microcontroller-based approaches to signal analysis introduce:
- non-deterministic latency
- contention with real-time control tasks
- inefficient sequential processing

---

### Proposed Solution

This project proposes a custom ASIC integrated within the Caravel platform that acts as an **edge-AI co-processor for VFD systems**.

The chip performs real-time analysis of motor signal windows using a fixed, hardware-accelerated neural inference pipeline.

System structure:
- External MCU/DSP: executes real-time motor control (FOC, PWM)
- ASIC (this work): performs continuous signal inference
- Decision layer: adjusts system behavior based on inference output

The ASIC operates in parallel with the control loop, enabling:
- deterministic, fixed-latency inference
- zero interference with control execution
- continuous monitoring of motor health

---

### Silicon Design Specifications

The design is constrained to the Caravel user project area (~10 mm², SKY130 process) and optimized for a single fixed workload.

#### NPU Configuration
- Architecture: fixed-function INT8 inference engine
- MAC array: 4×4 (16 MACs per cycle)
- Supported models: small 1D CNN / MLP

#### Model Parameters
- Input: 128-sample time-series window
- Convolution: 8 filters, kernel size 3
- Dense layer: 8 → 4 outputs
- Total compute: ~3056 MAC operations per inference

#### Performance
- Clock frequency: ~25 MHz (conservative target)
- Inference latency: ~191 cycles (~7.6 µs)
- Compatible with VFD control loops (5–20 kHz, 50–200 µs period)

#### Memory
- Input buffer: 128 bytes
- Weights: ~56 bytes
- Total on-chip memory: <1 KB

#### Interfaces
- Data input via SPI/I2C from external MCU
- Classification output via register interface

#### Design Characteristics
- Fully deterministic execution (fixed cycle count)
- No floating-point support (INT8 only)
- No on-chip training or reconfiguration
- Streaming dataflow architecture

---

### Outcome

The resulting chip provides a **low-latency, energy-efficient, and deterministic inference engine** for real-time motor signal analysis, enabling earlier fault detection and adaptive control in VFD systems under strict silicon constraints.