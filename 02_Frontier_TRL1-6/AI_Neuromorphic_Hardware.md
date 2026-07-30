---
title: "Sovereign AI Stack Architecture: Sino-Russian Hardware-Software Synthesis"
tags: [tech-sovereignty, risc-v, memristor, moe-models, mlir, china, russia, hardware-design, reram, neuromorphic]
trl_level: "TRL 1-6"
---

{% hint style="info" %}
**Executive Conceptual Summary**
This document formulates the system engineering and mathematical architecture for a **Non-CUDA / Non-x86 Sovereign AI Computing Platform**. Designed to operate under trade restrictions and export controls (EAR/FDPR), the platform establishes a Sino-Russian technical division of labor: **Russian Fundamental Mathematics** (topological compilation, non-Euclidean noise-tolerant quantization, differential geometry compilers) coupled with **Chinese Advanced Silicon Execution & Industrial Scaling** (SMIC DUV/N+2/N+3 lithography, [Chiplet Technology](../lib-1/Chiplet%20Technology.md) integration via UCIe, open [RISC-V](../lib-1/RISC-V.md) matrix extensions, and BEOL [Memristor Crossbar](../lib-1/Memristor%20Crossbar.md) integration).
{% endhint %}

---

## 1. Global Platform System Architecture

The target platform detaches the hardware/software stack from Western IP (x86, ARM, CUDA, ROCm) across three integrated layers. It matches math-heavy IR passes to low-yield or non-standard hardware topologies.

```
===================================================================================
                  SOVEREIGN MULTIMODAL APPLICATION LAYER
     [[Native Multimodal MoE]] Models (Text, Vision, Synthetic Radar/EW)
===================================================================================
                                       │
                                       ▼
+---------------------------------------------------------------------------------+
|   1. MATHEMATICAL & COMPILATION STACK (RU Mathematical Focus)                   |
|   - Riemannian Manifold Weight Mapping & Topological Noise Quantization        |
|   - Multi-Level IR Pipeline ([[MLIR Framework]] Custom Dialects: `sino_rvv`, `imc`)|
|   - Dynamic Fault-Tolerant Graph Compilers & Stochastic Graph Partitioning      |
+---------------------------------------------------------------------------------+
                                       │
                                       ▼
+---------------------------------------------------------------------------------+
|   2. HARDWARE & PACKAGING EXECUTION LAYER (CN Silicon Focus)                   |
|   +------------------------------------+  +-----------------------------------+ |
|   | Digital Compute Core (TRL 5–6)     |  | Analog Neuromorphic Engine (TRL 2–4)| |
|   | - Custom [[RISC-V]] (XiangShan /   |  | - [[Memristor Crossbar]] (OxRAM / | |
|   |   T-Head derived) Matrix Core      |  |   PCM BEOL Integration)           | |
|   | - RVV 1.0 + Custom Matrix ISA      |  | - Mixed-Signal GEMM Subsystem     | |
|   +------------------------------------+  +-----------------------------------+ |
|   +---------------------------------------------------------------------------+ |
|   | Packaging & Interconnect: 2.5D/3D [[Chiplet Interconnects]] (UCIe) +       | |
|   | Photonic/RoCEv2 Transport Fabric (No NVLink Dependency)                  | |
+---------------------------------------------------------------------------------+
```

---

## 2. Deep-Dive Subsystem Breakdown

### Subsystem A: RISC-V Vector & Custom ISA Matrix Accelerators (TRL 5–6)

The primary digital processing engine avoids proprietary IP by extending the open [RISC-V](../lib-1/RISC-V.md) ISA (RV64GCV) with domain-specific matrix units (`RVM` - RISC-V Matrix Extension).

```
                      +-----------------------------------+
                      |      PyTorch / ONNX Dialects      |
                      +-----------------------------------+
                                        │
                                        ▼
                      +-----------------------------------+
                      |      MLIR High-Level (`linalg`)   |
                      +-----------------------------------+
                                        │
                                        ▼
                      +-----------------------------------+
                      | MLIR Target Dialect (`sino_rvv`)  |
                      |  - Register Tiling (2D Block)     |
                      |  - Micro-Exponent Conversion      |
                      +-----------------------------------+
                                        │
                                        ▼
                      +-----------------------------------+
                      |  RISC-V Matrix ISA Execution Core |
                      |  - Vector Length (VLEN=2048)      |
                      |  - Matrix Accumulator (2D Array)  |
                      +-----------------------------------+
```

#### Custom ISA Architecture & Execution Pipeline
The core integrates a 2D Systolic [Systolic Array](../lib-1/Systolic%20Array.md) directly into the RISC-V pipeline. Compute units utilize a vector register width ($VLEN$) configured from $512$ up to $2048$ bits, with a vector data length ($DLEN = VLEN$).

*   **Matrix Register File**: 8 2D matrix tile registers (`mreg0`–`mreg7`), holding $16 \times 16$ sub-matrices.
*   **Precision Support**: Hardware decoding for FP16, BF16, INT8, FP4, and Micro-Exponent formats (MXFP6, MXFP4 with E2M1 and E3M0 exponent distributions).
*   **Instruction Set Primitives**:
    *   `vmatmul.mm mregA, mregB, mregC`: Multiplies two $16 \times 16$ 8-bit precision matrices, accumulating into a 32-bit register file in 16 clock cycles.
    *   `vquant.scale mregA, scale_factor`: Performs hardware-accelerated dynamic scaling for non-Euclidean quantized tensors.

```
// Custom MLIR Dialect Conversion Pass Example for RVM Matrix Loop Tiling
func.func @matrix_multiply(%arg0: memref<1024x1024xf16>, %arg1: memref<1024x1024xf16>, %arg2: memref<1024x1024xf32>) {
  // Tile loop iterations into 16x16 blocks matching hardware mreg bounds
  affine.for %i = 0 to 1024 step 16 {
    affine.for %j = 0 to 1024 step 16 {
      affine.for %k = 0 to 1024 step 16 {
        %tileA = sino_rvv.load_mreg %arg0[%i, %k] : memref<1024x1024xf16> -> !sino_rvv.mreg
        %tileB = sino_rvv.load_mreg %arg1[%k, %j] : memref<1024x1024xf16> -> !sino_rvv.mreg
        %tileC = sino_rvv.load_mreg %arg2[%i, %j] : memref<1024x1024xf32> -> !sino_rvv.mreg
        %res = sino_rvv.vmatmul %tileA, %tileB, %tileC : !sino_rvv.mreg
        sino_rvv.store_mreg %res, %arg2[%i, %j] : memref<1024x1024xf32>
      }
    }
  }
  return
}
```

#### Physical Bottlenecks & Silicon Fabrication Realities

{% hint style="warning" %}
**Physical Silicon Bottlenecks (SMIC N+2/N+3)**
1.  **Lithographic Constraints**: Due to EUV access blocks, production relies on Deep Ultraviolet (DUV) immersion lithography with Self-Aligned Quadruple Patterning (SAQP) at the SMIC N+2 ($7\text{ nm}$ equivalent) and N+3 nodes. This leads to higher mask counts, reduced wafer yield, and edge placement error (EPE) limits.
2.  **Memory Wall (No High-Tier HBM3e)**: Lacking TSMC advanced CoWoS and high-yield HBM3e interposers, memory bandwidth relies on 2.5D [Chiplet Technology](../lib-1/Chiplet%20Technology.md) with multi-channel LPDDR5X/GDDR6 arrays or high-density silicon interposers (SMIC OSAT capabilities).
{% endhint %}

---

### Subsystem B: In-Memory Neuromorphic Processor (TRL 1–4)

To bypass the von Neumann memory-wall, analog non-volatile memory crossbars perform matrix operations in the memory array itself.

```
                    Input Voltages V_i (from DAC Array)
                             │               │
                             ▼               ▼
                     ┌───────────────┴───────────────┐
  Wordline 1 (V_1) ─►│  G_11 (OxRAM)   G_12 (OxRAM)  │
                     │   [TiN/HfOx]     [TiN/HfOx]   │
  Wordline 2 (V_2) ─►│  G_21 (OxRAM)   G_22 (OxRAM)  │
                     └───────────────┬───────────────┘
                                     │               │
                                     ▼               ▼
                             Bitline 1 (I_1) Bitline 2 (I_2)
                        I_j = Σ (V_i * G_ij)  (Kirchhoff's Law)
                                     │               │
                                     ▼               ▼
                           [Transimpedance Amplifier (TIA)]
                                     │
                                     ▼
                          [Low-Power SAR-ADC Array]
```

#### Physics of Compute & Physical Non-Idealities
In-Memory Computing ([In-Memory Computing](../lib-1/In-Memory%20Computing.md)) with [Memristor Crossbar](../lib-1/Memristor%20Crossbar.md) arrays ($\text{HfO}_x/\text{TaO}_x$ OxRAM or $\text{Ge}_2\text{Sb}_2\text{Te}_5$ PCM) processes General Matrix Multiply ($\text{GEMM}$) operations using Ohm's Law and Kirchhoff's Current Law:

$$I_j = \sum_{i=1}^{N} V_i \cdot G_{ij}$$

Where $V_i$ represents wordline voltage, $G_{ij}$ is non-volatile memristive conductance, and $I_j$ is output bitline current.

##### Physical Non-Idealities & Mathematical Drift Models
Memristive conductances deviate from ideal linear weights due to physical dynamics:

1.  **Filamentary Oxygen Vacancy Drift**: Conductance degrades over time ($t$) following a power-law relationship:
    $$G(t) = G_0 \cdot \left(\frac{t}{t_0}\right)^{-\nu}$$
    where $\nu \approx 0.03 - 0.08$ is the drift coefficient.
2.  **Parasitic Line Resistance ($R_{\text{wire}}$)**: Wire resistance along bitlines causes an IR voltage drop, altering the effective voltage across deep crossbar cells:
    $$V_{i,j}^{\text{actual}} = V_i - I_{\text{line}} \cdot R_{\text{wire}}(i, j)$$

```
+-------------------------------------------------------------------+
|               CROSSBAR POWER CONSUMPTION BREAKDOWN                |
+-------------------------------------------------------------------+
| Analog Compute Array (Memristive MAC Operations): [ 12% Power ]   |
| Mixed-Signal Conversion (ADCs, DACs, TIAs):       [ 88% Power ]   |
+-------------------------------------------------------------------+
```

{% hint style="danger" %}
**ADC/DAC Energy Paradox**
While the analog matrix array executes matrix multiplication at $>100\text{ TOPS/W}$, converting incoming analog signals back to digital logic requires High-Speed SAR ADCs. **Mixed-signal converters consume 85–90% of total active silicon die area and operational power.**
{% endhint %}

#### Russian Mathematical Mitigation: Topological Noise-Tolerant Mapping
To maintain model accuracy with high physical noise ($\pm 20\%$ conductance drift) and low-resolution ADCs (e.g., 3-bit to 4-bit output quantization), weights are mapped onto Riemannian manifolds:

```
[Standard Model Weights] ──► Riemannian Manifold Embedding ──► Topological Invariant Vectors ──► Dynamic Memristor Mapping
```

1.  **Topological Invariant Mapping**: Models are trained by projecting weight matrices onto compact Riemannian manifolds where topological invariants (Betti numbers, persistent homology features) store continuous representations.
2.  **Noise Mitigation Equation**: The loss function incorporates a stochastic noise penalty matrix $\mathbf{\Sigma}_{\text{drift}}$ matching the physical memristive profile:
    $$\mathcal{L}_{\text{sovereign}} = \mathcal{L}_{\text{task}}(W) + \lambda \mathbb{E}_{\delta \sim \mathcal{N}(0, \mathbf{\Sigma}_{\text{drift}})} \left[ \|\nabla_W \mathcal{L}_{\text{task}}(W) \cdot \delta\|^2 \right]$$

---

### Subsystem C: Sovereign Native Multimodal MoE Engine (TRL 4–6)

The software architecture uses scalable [Native Multimodal MoE](../lib-1/Native%20Multimodal%20MoE.md) (Mixture-of-Experts) models to route task tokens (Text, Vision, Synthetic Radar/EW) to domain-specific execution experts.

```
                                [ Input Tokens ]
                                       │
                                       ▼
                       [ Gating Network Router W_g ]
                                       │
                 ┌─────────────────────┼─────────────────────┐
                 │ Top-k Routing (k=2) │ Non-Euclidean Scale │
                 └─────────────────────┬─────────────────────┘
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        ▼                              ▼                              ▼
[ Expert 1: Language ]      [ Expert 2: Vision/Patch ]     [ Expert N: Signal/EW ]
(RISC-V Vector Cores)        (RISC-V Vector Cores)        ([[Memristor Crossbar]])
```

#### Expert Routing Formulation & Top-k Gating
The routing network evaluates incoming token representation $x$ and distributes activation vectors to selected experts $E_i(x)$:

$$y = \sum_{i \in \text{TopK}(G(x), k)} G(x)_i \cdot E_i(x)$$

$$G(x) = \text{Softmax}\left(\text{TopK}\left(x \cdot W_g + \epsilon, k\right)\right)$$

Where $W_g$ is the trainable gate parameter matrix, $\epsilon \sim \mathcal{N}(0, \sigma^2)$ is standard Gaussian noise for load balancing, and $k=2$ experts per token.

#### Scaling Model Execution Without High-Bandwidth NVLink
Lacking high-speed interconnects like Nvidia NVLink ($900\text{ GB/s} - 1.8\text{ TB/s}$), clusters handle Expert-Parallel All-to-All communication bottlenecks via hardware/compiler co-design:

```
+-------------------------------------------------------------------+
|               NON-NVLINK CLUSTER INTERCONNECT STACK               |
+-------------------------------------------------------------------+
| Multi-Rail RoCEv2 (100G/400G) + Custom Hardware Transport Engines  |
| Top-K Expert Allocation Constraints (Limits Inter-Node Transfers) |
| Async Gradient & Token Pipelining via Predictive Graphs           |
+-------------------------------------------------------------------+
```

*   **Multi-Rail RoCEv2 Fabric**: Multi-rail RDMA over Converged Ethernet (RoCEv2) with custom priority-based flow control (PFC) protocols.
*   **Locality-Constrained Expert Placement**: The compiler assigns topological expert pairs to the same silicon package connected via 2.5D [Chiplet Interconnects](../lib-1/Chiplet%20Interconnects.md) (UCIe) to minimize cross-node communication overhead.
*   **Predictive Token Pipelining**: Compilers analyze multi-head attention graph flows to pre-fetch network tokens before standard gating resolution finishes, concealing network latency behind matrix computation phases.

---

## 3. Geoeconomics, Patent Thickets, and IP Resilience

```
                               ┌───────────────────────────┐
                               │ Export Control Barriers   │
                               │  (EAR / FDPR Regulations) │
                               └─────────────┬─────────────┘
                                             │
                                             ▼
         ┌───────────────────────────────────┴───────────────────────────────────┐
         │                                                                       │
         ▼                                                                       ▼
┌────────────────────────────────┐                             ┌────────────────────────────────┐
│   X86 / ARM / CUDA IP BLOCKS   │                             │   PATENT THICKETS (US/EU/JP)   │
│ Western hardware licenses and  │                             │ Proprietary IMC circuits,      │
│ software runtimes prohibited   │                             │ HBM and Attention patents      │
└────────┬───────────────────────┘                             └────────┬───────────────────────┘
         │                                                              │
         └───────────────────────────────┬──────────────────────────────┘
                                         │
                                         ▼
                        ┌─────────────────────────────────┐
                        │   SINO-RUSSIAN SOVEREIGN STACK  │
                        │ - Open [[RISC-V]] Core Substrate │
                        │ - Custom Open ISA Matrix Units  │
                        │ - Non-Infringing Math Mapping   │
                        │ - Open-Source MLIR Dialects     │
                        └─────────────────────────────────┘
```

1.  **Navigating the Foreign Direct Product Rule (FDPR)**: Sanction controls target Western x86/ARM hardware IP along with proprietary parallel computing frameworks like CUDA. The open [RISC-V](../lib-1/RISC-V.md) instruction set provides immunity from unilateral export revokations.
2.  **Mitigating Western Patent Thickets**: Strategic design relies on alternate open standards and mathematically equivalent matrix operations. Releasing foundational compilation logic under permissive, non-aligned regulatory frameworks (e.g., RISC-V International based in Switzerland) protects code distribution across global markets.

---

## 4. Comprehensive Technology Readiness & Physical Barriers Matrix

| Technology Subsystem | TRL | Primary Physical / Infrastructure Bottleneck | Strategic Compiler & Mathematical Mitigation | Sovereign Target Specification |
| :--- | :--- | :--- | :--- | :--- |
| **RISC-V Vector & Matrix Cores** | **TRL 5–6** | EDA tool access limitations ($<5\text{ nm}$); DUV multi-patterning yield degradation at SMIC N+2/N+3. | [MLIR Framework](../lib-1/MLIR%20Framework.md) vectorization passes; custom tiling for missing hardware instructions. | $1.5\text{ GHz}$, $2048\text{-bit } VLEN$, $32\text{ TFLOPS}$ (BF16) per core block. |
| **Memristor IMC Arrays** | **TRL 2–4** | OxRAM/PCM conductance drift; high power footprint of peripheral ADC/DAC circuits. | Riemannian Manifold weight projection; dynamic topological noise-tolerant quantization. | $>50\text{ TOPS/W}$ raw compute density at sub-4-bit effective precision. |
| **Sovereign Native MoE Engine** | **TRL 4–6** | High latency during All-to-All communication over Non-NVLink network protocols. | Top-k locality-constrained routing; dynamic token pre-fetching in graph compiler. | Scalable to 100B+ parameter models over standard RoCEv2 clusters. |
| **UCIe Chiplet Interconnects** | **TRL 3–4** | Substrate routing density limitations; package yield loss on multi-die modules. | Error-correcting die-to-die transport protocols; redundant link mapping. | $2\text{ Tbps/mm}$ shoreline bandwidth using 2.5D interposer packaging. |

---

## 5. Self-Assessment Case Studies & Interactive Analysis

### Case 1: The Mixed-Signal ADC Bottleneck in Neuromorphic Edge Hardware

{% hint style="info" %}
**Practical Engineering Scenario**
A mixed-signal engineering group fabricates a 40nm $\text{HfO}_x$ ReRAM [Memristor Crossbar](../lib-1/Memristor%20Crossbar.md) engine for real-time edge processing. The internal analog array achieves a raw energy efficiency of $60\text{ TOPS/W}$. However, system measurements reveal that system-level energy efficiency drops to $6\text{ TOPS/W}$ once data crosses the mixed-signal boundary.

**Question**: Identify the primary causes of this efficiency drop and formulate a software/compilation strategy to recover compute performance without modifying the physical die layout.
{% endhint %}

```
+---------------------------------------------------------------------------------+
| ANALOG CORE EFFICIENCY: 60 TOPS/W                                               |
| [=============================================================================] |
| SYSTEM EFFICIENCY (WITH ADCs/DACs): 6 TOPS/W                                    |
| [=======]                                                                       |
+---------------------------------------------------------------------------------+
```

{% hint style="success" %}
**Diagnostic & Architectural Resolution**
1.  **Root Cause Identification**:
    *   **The Mixed-Signal Conversion Wall**: High-resolution Successive Approximation Register (SAR) ADCs require exponential capacitive area scaling ($C \propto 2^N$ for $N$ bits). At 8-bit precision, energy consumption is dominated by peripheral capacitor switching rather than array matrix multiplication.
    *   **Static Wire Loss**: Uncompensated IR drops across long bitlines forces the driver to run higher source voltages, lowering overall system efficiency.
2.  **Compiler & Mathematical Recovery Vector**:
    *   **1-Bit Dynamic Delta-Sigma Converters**: Re-architect compilation passes to target single-bit Comparator/ADC passes, replacing multi-bit SAR-ADCs with high-frequency 1-bit bitstream integration.
    *   **Topological Algorithmic Sparsification**: The mathematical compiler enforces an $85\%$ weight-and-activation sparsity constraint. By gating input voltages $V_i = 0$ for zero-value tokens, driver DACs remain inactive during sparse cycles, raising system efficiency back above $>28\text{ TOPS/W}$.
{% endhint %}

---

### Case 2: Mitigating Inter-Node Latency in Non-NVLink MoE Training Clusters

{% hint style="info" %}
**System Scaling Scenario**
You are training a $120\text{B}$ parameter [Native Multimodal MoE](../lib-1/Native%20Multimodal%20MoE.md) network using a cluster of 1,024 [RISC-V](../lib-1/RISC-V.md) Matrix compute nodes connected via standard dual-rail 100G RoCEv2 switches. Execution trace logging indicates that Matrix Cores spend $68\%$ of active processing cycles idling during Expert Routing phases (`All-to-All` collective communication operations).

**Question**: Formulate a dynamic compiler mapping strategy and algebraic gating algorithm to reduce network latency and restore matrix unit utilization to $>70\%$.
{% endhint %}

```
+---------------------------------------------------------------------------------+
| MATRIX CORE UTILIZATION BEFORE OPTIMIZATION: 32%                                |
| [========================]                                                      |
| TARGET SYSTEM UTILIZATION AFTER OPTIMIZATION: >70%                              |
| [========================================================================]      |
+---------------------------------------------------------------------------------+
```

{% hint style="success" %}
**System Architectural Resolution**
1.  **Root Cause Identification**:
    *   Standard top-$k$ gating randomly distributes active experts across physical nodes. Without high-speed NVLink interconnects, point-to-point switch hops cause packet loss, network congestion, and execution latency stalls.
2.  **System Engineering Mitigation Plan**:
    *   **Topological Graph Partitioning**: Re-architect the router compiler pass using a hierarchical routing model:

$$G_{\text{hierarchical}}(x) = \text{Softmax}\left(W_{\text{cluster}} \cdot x\right) \otimes \text{Softmax}\left(W_{\text{local}} \cdot x\right)$$

*   **Execution Strategy**:
    1.  *Stage 1*: The router assigns $80\%$ of active token pathways to local experts located on the same physical chiplet package linked via high-speed [Chiplet Interconnects](../lib-1/Chiplet%20Interconnects.md) (UCIe).
    2.  *Stage 2*: Cross-node transfers are restricted to a pre-allocated fraction of the overall network bandwidth, dropping outlier tokens beyond a strict capacity factor ($\text{Capacity Factor} = 1.1$).
    3.  *Stage 3*: Overlap compute and communication pipelines by configuring the [MLIR Framework](../lib-1/MLIR%20Framework.md) engine to execute localized self-attention layers while async RoCEv2 transfers route non-local tokens across the background network. This restores core compute utilization to $74\%$.
{% endhint %}

---

## 6. Strategic Implementation Roadmap

```
2026-2027 (TRL 1 -> 3)            2028-2029 (TRL 3 -> 5)            2030+ (TRL 5 -> 6)
┌────────────────────────┐        ┌────────────────────────┐        ┌────────────────────────┐
│ - MLIR sino_rvv Passes │───────►│ - SMIC N+2 Tapeouts    │───────►│ - Scale Multi-Node     │
│ - Math Noise Models    │        │ - 2.5D UCIe Packaging  │        │   MoE Data Centers     │
│ - OxRAM Bit-Cell Validation     │ - Analog Neuromorphic   │        │ - Fully Autonomous     │
│                        │        │   Co-Processor Edge Cores│        │   Compiler Runtime     │
└────────────────────────┘        └────────────────────────┘        └────────────────────────┘
```

{% hint style="info" %}
**Key Execution Priorities**
1.  **Standardize Open MLIR Compiler Passes**: Prioritize the development of unified open-source compilation drivers within the [MLIR Framework](../lib-1/MLIR%20Framework.md) (`sino_rvv` and mixed-signal `imc` target dialects) to eliminate reliance on closed toolchains.
2.  **Target Neuromorphic IMC at SWaP-Constrained Edges**: Restrict analog [Memristor Crossbar](../lib-1/Memristor%20Crossbar.md) hardware deployments to Size, Weight, and Power (SWaP) edge applications (drones, RF signal acquisition, localized sensors) where high hardware noise tolerances are mathematically viable.
3.  **Industrialize Advanced 2.5D Interposer Packaging**: Standardize die-to-die interconnect interfaces using modular [Chiplet Interconnects](../lib-1/Chiplet%20Interconnects.md) (UCIe) over high-density silicon interposers to bypass single-die lithographic yield limits at advanced nodes.
{% endhint %}

---
*Cross-References & Related Nodes*:
*   [RISC-V Vector Extensions](../lib-1/RISC-V%20Vector%20Extensions.md)
*   [In-Memory Computing](../lib-1/In-Memory%20Computing.md)
*   [Memristor Crossbar](../lib-1/Memristor%20Crossbar.md)
*   [Native Multimodal MoE](../lib-1/Native%20Multimodal%20MoE.md)
*   [MLIR Framework](../lib-1/MLIR%20Framework.md)
*   [Chiplet Interconnects](../lib-1/Chiplet%20Interconnects.md)
*   [Systolic Array](../lib-1/Systolic%20Array.md)
*   [ReRAM](../lib-1/ReRAM.md)
*   [Neuromorphic Computing](../lib-1/Neuromorphic%20Computing.md)