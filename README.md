# Edge-AI Co-Processor for Intelligent VFD Systems


## 1.0 Summary

### 1.1 Problem

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

### 1.2 Proposed Solution

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

### 1.3 Silicon Design Specifications

The design is constrained to the Caravel user project area (~10 mm², SKY130 process) and optimized for a single fixed workload.

#### 1.3.1 NPU Configuration
- Architecture: fixed-function INT8 inference engine
- MAC array: 8×8 (64 MACs per cycle)
- Supported models: small 1D CNN / MLP

#### 1.3.2 Model Parameters
- Input window: 256-sample INT8 per pipeline
- Convolution: 16 filters, kernel size 3
- Dense layer: 16 → 4 outputs per pipeline
- Total compute per inference: ~12,288 MACs per pipeline
- Two pipelines → ~24,576 MACs per cycle window

#### 1.3.3 Performance
- Clock frequency: ~25 MHz (conservative target)
- Inference latency: ~191 cycles (~7.6 µs)
- Compatible with VFD control loops (5–20 kHz, 50–200 µs period)

#### 1.3.4 Memory
- Input buffers: 1 KB per pipeline → 2 KB total
- Weights: 4 KB per pipeline → 8 KB total
- Intermediate activations + scratchpad: 6 KB
- Total on-chip SRAM: ~16 KB

#### 1.3.5 Interfaces
- Data input via SPI/I2C from external MCU
- Classification output via register interface

#### 1.3.6 Design Characteristics
- Fully deterministic execution (fixed cycle count)
- No floating-point support (INT8 only)
- No on-chip training or reconfiguration
- Streaming dataflow architecture

---

### 1.4 Outcome

The resulting chip provides a **low-latency, energy-efficient, and deterministic inference engine** for real-time motor signal analysis, enabling earlier fault detection and adaptive control in VFD systems under strict silicon constraints.

## 2.0 Detailed Chip Architecture

The ASIC is designed as a streaming, fixed-function inference pipeline optimized for deterministic execution and minimal memory usage.

The architecture consists of the following tightly coupled modules:

1. Input Interface
2. Window Buffer
3. Preprocessing Unit
4. Neural Processing Unit (NPU)
5. Control Unit (FSM)
6. Output Interface
7. Management Core
8. End-to-End Dataflow

### 2.1 Input Interface

The input interface receives time-series samples from an external MCU via SPI.

Responsibilities:
- Accept 8-bit sampled data
- Handle streaming input at fixed rate
- Write incoming samples into the window buffer

Assumptions:
- Sampling is performed externally (ADC on MCU)
- Data arrives at a rate compatible with control loop frequency (e.g. 5–20 kHz)

Design:
- SPI slave module
- Simple register-mapped write interface
- Backpressure via ready/valid signaling


### 2.2 Window Buffer

The window buffer stores a sliding window of 128 samples used as input to the inference pipeline.

Design:
- FIFO or circular buffer
- Depth: 128 samples (8-bit each)
- Total size: 128 bytes

Operation:
- New samples overwrite oldest samples (sliding window)
- Once buffer is full, inference is triggered periodically

Key property:
- No large memory allocations
- Continuous streaming operation

### 2.3 Preprocessing Unit

- Lightweight normalization of inputs: mean subtraction, scaling, clipping.
- Implemented with combinational arithmetic and small registers.
- Ensures input consistency without large multipliers or floating-point units.

### 2.4 Neural Processing Unit (NPU)
- **Two parallel pipelines**, each performing a fixed 1D CNN + dense layer.
- **MAC array:** 8×8 (64 MAC units), INT8 arithmetic, fully pipelined.
- **Model per pipeline:**
  - Input: 256-sample time series
  - Convolution: 16 filters, kernel size 3
  - Dense layer: 16 → 4 outputs
  - Total MACs: ~12,288 per inference per pipeline
- **Dataflow:** streaming from buffers → preprocessing → convolution → dense → output
- Deterministic execution: 192 cycles per inference → ~7.7 µs per pipeline

### 2.5 Control Unit (FSM)
- Finite state machine coordinating inference execution:
  - IDLE → LOAD_WINDOW → CONV_COMPUTE → DENSE_COMPUTE → OUTPUT_WRITE → DONE
- Schedules MAC array operations, data movement, and buffer access
- Fully deterministic; no dynamic branching or variable memory access

### 2.6 Output Interface
- Memory-mapped registers expose inference results to the MCU
- Optional interrupt signals indicate inference completion
- Outputs: classification results and optional confidence scores

### 2.7 Management Core

The Management Core oversees all high-level operations of the ASIC. It does not perform MAC computations but ensures coordination, configuration, and system health. Key responsibilities:

- **System Initialization**
  - Configures I/O interfaces (SPI/I2C)
  - Sets clock gating and NPU enable signals
  - Initializes buffers and weights

- **Mode Control**
  - Switch between operational modes (idle, inference, diagnostics)
  - Supports single-inference or continuous streaming

- **Configuration Registers**
  - Holds parameters such as:
    - Input buffer length
    - Inference triggers
    - Optional scaling or normalization factors

- **Monitoring & Fault Detection**
  - Monitors internal signals (buffer overflow, NPU busy)
  - Signals external MCU in case of error

- **Trigger Coordination**
  - Decides when the FSM should start inference
  - Can schedule periodic inference based on external sampling rate

- **Power Management**
  - Can gate clocks to unused modules
  - Reduces energy when idle

### 2.8 End-to-End Dataflow
1. MCU streams motor signals → Input Interface
2. Circular buffers store data for two parallel pipelines
3. Management Core monitors buffer readiness → triggers FSM
4. Preprocessing normalizes input windows
5. NPU executes convolution + dense layers in two pipelines
6. Results written to output registers
7. MCU reads outputs to adjust VFD control or trigger alarms