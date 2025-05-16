# Apple Neural Engine (ANE) Documentation

This document provides a comprehensive analysis of the Apple Neural Engine based on the reverse-engineering work in tinygrad's `extra/accel/ane` directory.

## Overview

The Apple Neural Engine (ANE) is a specialized hardware accelerator for neural network operations in Apple devices. It first appeared in the A11 Bionic chip and has been included in subsequent Apple silicon. The ANE is designed to efficiently execute neural network workloads with low power consumption.

The implementation in tinygrad represents a reverse-engineered approach to interfacing with ANE directly, bypassing Apple's official frameworks and providing low-level access to the hardware.

## System Architecture

### Hardware Characteristics

- ANE is primarily a DMA engine optimized for convolutions
- Features 16 cores (likely a 16-wide Kernel DMA engine)
- Has a 4MB L2 cache that must be manually managed
- Claimed performance of 11 TOPS (possibly 32x32 MAC at ~335 MHz)
- The L2 "cache" is manually managed and only applies to input/output, not weights
- All strides must be a multiple of 0x40 bytes

### Software Stack

- **Kernel Driver**: `AppleH11ANEInterface` (requires `com.apple.ane.iokit-user-access` entitlement)
- **Helper Processes**:
  - `/usr/libexec/aned` - ANE daemon
  - `ANECompilerService` - Handles compilation tasks
- **Framework Stack**:
  - **Espresso**: Contains `ANECompilerEngine`
  - **AppleNeuralEngine**: Objective-C interface called by Espresso
    - **ANEServices**: Communication with the device
    - **ANECompiler**: Compiles plist into hwx file

### Normal Flow for ANE Usage

```
    Keras/ONNX model
          |
          |   1_build
          | (coremltools, open source)
          v
      CoreML model
          |
          |   (Espresso)
          v
       net.plist
          |
          |   2_compile
          | (AppleNeuralEngine, ANECompiler)
          v
       model.hwx
          |
          |   3_run
          | (AppleNeuralEngine, ANEServices, AppleH11ANEInterface)
          v
 <run on neural engine>
```

## Data Handling

### Tensor Structure

ANE works with 5D tensors:
- Column (width)
- Row (height)
- Plane (channels)
- Depth
- Group (batch)

### Supported Data Types

- UInt8
- Int8
- Float16 (Float32 inputs may be automatically converted to Float16)

## Operation Structure

Each ANE operation is 0x300 bytes in size and has several components:

### 1. Header Section (0x0-0x2C)
- 0x1C: Next operation offset
- 0x20: Output address

### 2. KernelDMASrc (0x2C-0x128)
- 16 output channels in parallel
- 0x34-0x74: Channel usage flags (0x80 | 1 if used)
- 0x74-0xB4: Channel data offsets
- 0xB4-0xF4: Channel data lengths

### 3. Common (0x128-0x16C)
- Input/output dimensions and types
- Kernel size and padding configuration
- Batch size

### 4. TileDMASrc (0x16C-0x1E0)
- Input memory layout (strides for rows, planes, depth, batch)
- Input interleave configuration

### 5. L2 (0x1E0-0x22C)
- Input/output interleave settings
- Channel stride configurations

### 6. NE (0x22C-0x240)
- Bias and scale scalar values
- Neuron activation type

### 7. TileDMADst (0x258-0x274)
- Output memory layout (strides for rows, planes, depth, batch)
- Output interleave configuration

## Supported Operations

ANE supports a wide range of neural network operations:

### Basic Operations
- CONV (Convolution)
- POOL (Pooling)
- EW (Element-wise operations)
- CONCAT (Concatenation)
- RESHAPE
- TRANSPOSE
- MATRIX_MULT (Matrix multiplication)

### Advanced Operations
- SCALE_BIAS
- TERNARY_DYNAMIC_GOC
- BINARY_DYNAMIC_GOC
- ACTIVATION
- SCALED_EW (Scaled element-wise)
- BROADCAST
- SOFTMAX
- INSTANCE_NORM
- L2_NORM

### Fused Operations
- PEFUSED_ELEMENTWISE
- PEFUSED_POOL
- PEFUSED_GOC
- NEFUSED_CONV
- NEFUSED_POOL
- NEFUSED_EW
- NEFUSED_BYPASS

## Activation Functions

ANE supports numerous activation functions:
- NONE
- RELU
- SIGMOID (standard and high precision)
- TANH
- CLAMPED_RELU
- PRELU
- SQRT/RSQRT
- LOG2/EXP2
- ELU
- SIGN
- CUSTOM_LUT (Custom lookup table)
- And many more (over 30 types)

## Implementation in tinygrad

The ANE implementation in tinygrad is structured into three main components:

### 1. Build (`1_build/`)

The build component creates CoreML models that can be later compiled for ANE:

- **`coreml_ane.py`**: Creates a simple CoreML model using `coremltools`
  - Defines input and output features
  - Creates a neural network builder
  - Adds operations (elementwise, inner product, etc.)
  - Compiles and saves the model

### 2. Compile (`2_compile/`)

The compilation component transforms CoreML models into a format that can run on ANE:

- **`ane.py`**: Core implementation for interfacing with ANE
  - Defines `ANE_Struct` - memory layout of ANE operations
  - Implements `ANETensor` class for tensor management
  - Provides the `ANE` class with methods for compilation and execution

- **`compile.mm`**: Interfaces with Apple's ANECompiler framework
  - Sets up compilation parameters
  - Calls `ANECCompile` to compile network plists into HWX files

- **`hwx_parse.py`**: Analyzes and compares compiled HWX files
  - Parses Mach-O formatted HWX files
  - Extracts sections and symbols
  - Compares different HWX files

- **`plists/` and `simple/`**: Contains property list files defining different operations

### 3. Run (`3_run/`)

The run component executes compiled models on ANE:

- **`h11ane.h`**: Defines the interface to the ANE hardware
  - Declares structures and classes for ANE interaction
  - Defines the `H11ANEDevice` and `H11ANEDeviceController` classes

- **`test.mm`**: Demonstrates how to execute compiled models on ANE
  - Initializes the ANE device
  - Loads a compiled HWX model
  - Creates input/output buffers using IOSurface
  - Executes the model and retrieves results

## Compiler Internals

The ANE compiler follows this internal call stack:

```
ANECCompileProcedure
  ZinAneCreateIr
    ZinParseNeuronUnit
  ZinAneCoreCompile
    ZinAneCodeGeneration
      ZinIrCodegenHandleKernels
      ZinIrTargetH13::CodegenTds
        ZinIrCacheHintTable
        ZinIrCodegenHandleTds_v7
          ZinIrCodegenHandleTdsMakeList<7u>
            ZinAneInstruction
            ZinAneTd<7u>::HandleEngineLayer
              ZinIrCodegenHandleTds<7u>
```

### Compiler Flags

Actual compiler flags used in ANE compilation:

```
zin_ane_compiler v4.2.1
	-t h13
	--fdram-allocator=ffreuse
	--fdram-tensor-priority=sizethenliverange
	--fl2-allocator=ffreuse
	--fl3-allocator=ffreuse
	--fl2-cache-mode=resident
	--fsignature=ident
	--memcache-size=4194304
	--fspatial-split=disabled
	--fkernel-rewind=enabled
```

## Security and Entitlements

ANE access requires special entitlements that are not normally available to third-party applications. Several approaches to bypass Apple's security restrictions:

1. Disabling `amfid` (Apple Mobile File Integrity Daemon)
2. Patching `amfid` to allow restricted entitlements
3. Using provisioning profiles
4. Patching the ANE kernel extension

## HWX File Format

- HWX files are Mach-O formatted binaries
- Operations start at offset 0x4000
- Each operation is 0x300 bytes in size
- The file contains the compiled program that can be directly executed by the ANE hardware

## Memory Management

- ANE has sophisticated memory management with different allocators for DRAM, L2, and L3 cache
- The L2 "cache" is manually managed and only applies to input/output
- Cache hints (ALLOC, NOALLOC, DROP, DEPRI) are used to optimize memory usage
- IOSurface is used for efficient memory sharing between CPU and ANE

## Key Insights and Takeaways

1. **Proprietary Architecture**: ANE uses a proprietary architecture with custom operation codes and memory layouts.

2. **DMA-Centric Design**: At its core, ANE is a sophisticated DMA engine optimized for convolutions and other neural network operations.

3. **Memory Optimization**: The hardware has been designed with careful attention to memory bandwidth, using a manually managed L2 cache and various stride optimizations.

4. **Compilation Pipeline**: The compilation process involves multiple stages and transformations, from high-level neural network descriptions to low-level hardware instructions.

5. **Security Restrictions**: Apple has placed significant security barriers around ANE access, requiring special entitlements and workarounds for direct hardware access.

6. **Operation Support**: Despite being specialized hardware, ANE supports a wide range of neural network operations, including basic convolutions, element-wise operations, and more complex fused operations.

7. **Execution Model**: ANE operations are executed asynchronously with message-based completion notification, allowing for efficient parallel processing.

8. **Data Type Limitations**: The hardware primarily works with Float16 and integer types, with automatic conversion from Float32 when necessary.

9. **Tensor Dimensionality**: ANE's support for 5D tensors allows for efficient representation of complex neural network structures.

10. **Reverse Engineering Challenges**: The implementation in tinygrad demonstrates the significant effort required to reverse-engineer proprietary hardware interfaces without official documentation.

## Conclusion

The ANE implementation in tinygrad provides valuable insights into Apple's neural engine architecture and API. By reverse-engineering the interface, this code enables direct access to ANE's capabilities without relying on Apple's high-level frameworks.

This work represents a significant effort in understanding a proprietary hardware accelerator, offering the potential for more efficient and customized neural network implementations on Apple platforms.


# Apple Neural Engine (ANE) / Apple NPU – Reverse-Engineering Notes

This document consolidates **all insights, take-aways, and technical revelations** extracted from **every single file** in `extra/accel/ane` inside the tinygrad repository.  It is intended both as learning material and as a map for anyone wishing to extend tinygrad’s experimental ANE back-end or to perform further research on Apple’s NPU.

---

## Directory Map

```
extra/accel/ane/
├── 1_build/           # Produce CoreML models
├── 2_compile/         # Transform models to ANE-executable HWX
├── 3_run/             # Execute HWX directly on the device
├── amfi/              # Entitlement / code-signing bypass helpers
├── lib/               # Stand-alone ANE loader + helpers (no tinygrad)
├── ops/               # Pre-compiled HWX kernels for inspection
├── tinygrad/          # tinygrad integration layer
├── README.md          # Polished, user-friendly overview
└── README.old         # Raw field notes while reversing
```

Below every file is listed, grouped by directory, followed by global findings.

---

## 1_build  — Creating Test Models

| File | Purpose | Key Learnings |
|------|---------|--------------|
| `.gitignore` | Ignore `DerivedData/` etc. | — |
| `coreml_ane.py` | Build a **64-element element-wise ADD** network with *coremltools* and run it, forcing execution on ANE. | Shows the *minimum viable* CoreML graph that triggers ANE; demonstrates that ANE accepts FP16 math when invoked through CoreML. |
| `run.swift` | Native Swift wrapper to load & run the created `.mlmodel`. | Confirms identical behaviour from Swift/Obj-C layer. |
| `test.mlmodel` | Generated model for manual inspection. | Mach-O package containing the CoreML spec. |

**Take-away:** *coremltools* is sufficient to feed ANE, but we still rely on Apple’s protected CoreML → Espresso → ANECompiler chain. The later directories show how to break free from that chain.

---

## 2_compile  — Turning Plists into HWX

### Top-level scripts

| File | Purpose / Functionality | Insights |
|------|------------------------|----------|
| `ane.py` | Pure-Python loader wrapping a private `libane.dylib`. Exposes `ANE.compile`, `ANE.run`, tensor helpers, register pack/unpack routines, and a pretty printer for ANE registers. | • **Operation layout** table (`ANE_Struct`) is the most accurate public description of an ANE TD (Tile Descriptor).<br>• Demonstrates that each op is exactly **0x300 bytes**.<br>• Confirms three data types: **UInt8 / Int8 / FP16**.<br>• Shows register bitfields by name, built from `aneregs.json`. |
| `aneregs.json` | 3-tuple `[byte, bit, width]` for **every single undocumented ANE register**. | Crucial for decoding TDs and for diffing compiler output. |
| `compile.mm` | Objective-C++ harness that calls the private **ANECompiler**’s symbol `ANECCompile`. Builds `model.hwx` from any `net.plist`. | Reveals the exact CoreFoundation dictionary keys the compiler expects (`InputNetworks`, `TargetArchitecture`, etc.). |
| `compile.m` | C wrapper used while experimenting with the same symbol. | Alternative stub. |
| `compile.sh` | Command-line convenience. | Documents required environment (codesign / entitlements). |
| `dcompile.py` | Dead-simple Python that spawns `compile.mm` repeatedly. | Automates batch experiments. |
| `hwx_parse.py` | Heavyweight diff/visualiser for two HWX binaries. Uses `macholib`, colorises differences, and maps bytes back to register names via `ANE.debug`. | Showed that **compiler emits relocation tables** for weights and IOSurfaces; helped reverse TD structure. |
| `struct_recover.py` | Script used to brute-force unknown struct offsets. | Assisted building `ANEREGS`. |

### Assets

Weights (`*.weights`) and sample net descriptors (`net.plist`) were captured during CoreML → Espresso compilation.  They allow offline reproduction of compiler behaviour.

`plists/` holds hand-written minimal plists for each op-code; `simple/` variants combine two or more ops to observe fused behaviour.

**Insights obtained from plists:**
- Mapping from high-level layer types to **Zin op-codes** (see README.old table).
- Observation that the compiler **fuses** kernel + activation + bias when possible.
- Confirmation that **Input/Output strides must be multiples of 0x40 bytes**.

---

## 3_run  — Talking to the Kernel Driver

| File | Purpose | Discoveries |
|------|---------|------------|
| `h11ane.h` | Re-created C++ header for **ANEServices** classes *H11ANEDevice* / *H11ANEDeviceController*. | Lists every user-client method name (power, logging, firmware, etc.).  Confirms two usage modes: `UsageCompile` (needs debugger) and `UsageWithProgram`. |
| `test.mm` | Full example that: 1) powers on ANE, 2) loads `model.hwx`, 3) allocates IOSurfaces, 4) sends `ANE_ProgramSendRequest`, 5) reads back result. | Shows the **Mach port IPC** pattern: async completion arrives on a receive port.  Also documents the unnamed 0x1000-word request struct. |
| `build.sh` | clang++ compile convenience. | Requires entitlements + `-framework IOSurface`. |
| `entitlements.xml` | Minimal XML granting `com.apple.ane.iokit-user-access`. | Must be injected via codesign or AMFI patch. |

---

## amfi  — Bypassing Code-Signing

| File | Description | Key Point |
|------|-------------|-----------|
| `new_patch.py` | LLDB script that **patches `/usr/libexec/amfid` in memory**, nop-ing the entitlement check. | Allows unsigned Python / clang binaries to access ANE without rebooting with `amfi_get_out_of_my_way`. |

---

## lib  — Standalone Loader (Not tinygrad-specific)

This sub-project appeared earlier in the research timeline and duplicates some functionality in `2_compile` + `3_run` but with a cleaner interface.

| File | Notes |
|------|------|
| `ane.mm` | Objective-C++ wrapper around `libane.dylib`; elegant C++ `Tensor` & `Program` classes. |
| `ane.py` | Pure python binding identical in spirit to `2_compile/ane.py` but without TD pretty-printer. |
| `aneregs.json` | Earlier version of the register map (superseded by the one in `2_compile`). |
| `build.sh` | Compiles `ane.mm` into `libane.dylib`. |
| `entitlements.xml` | Copy of minimal entitlement. |
| `h11ane.h` | Older, smaller extract of the C++ header. |
| `sign_python.sh` | Convenience codesign helper for the python interpreter itself. |
| `testconv.py` | End-to-end test: builds a 3×3 conv TD in python, uploads via `ane.py`, and verifies output. Shows manual TD creation without the compiler. |

---

## ops  — Pre-compiled Reference Kernels

Six `.hwx` blobs produced by Apple’s compiler.  Used to sanity-check the home-grown compiler and to diff against hand-written TDs.

| File | Op | Use |
|------|----|-----|
| `conv.hwx` | 3×3 FP16 convolution | Baseline for TD layout
| `relu.hwx` | Neuron only | Verified NE section
| `sigmoid.hwx` | Non-linear LUT | Verified custom LUT behaviour
| `concat.hwx`, `sum.hwx` | Multi-input TDs | Showed dual DMA source streams
| `gemm.hwx` | Large GEMM | Stress-tested row/plane strides

---

## tinygrad  — Integration Layer

| File | Purpose |
|------|---------|
| `ops_ane.py` | Implements tinygrad *LazyOp* → ANE translation for a subset of ops (conv, add, relu, gemm).  Picks between pre-compiled TDs in `ops/` or dynamically builds them via `2_compile/ane.py`. |

---

## README Files

| File | Nature | Highlights |
|------|--------|-----------|
| `README.md` | Clean public overview | Introduces high-level tensor layout, supported dtypes, manual L2 cache, 16-core figure, normal use-case flow, tips for extracting private frameworks. |
| `README.old` | **Raw research diary** | ‑ Full compiler backtraces<br>- Exact TD offset map (basis for `ANE_Struct`)<br>- List of every **Zin op-code** (0-48)<br>- List of **activation IDs** (0-31)<br>- Cache-hint enum (0-3)<br>- Compiler command-line printed by `aned` in dmesg.<br>- Notes on data-type coercions (FP32→FP16). |

---

## Global Technical Insights

1. **Tile Descriptor (TD) = 0x300 bytes** with seven logical sections (Header, KernelDMA, Common, TileDMA-Src, L2, NE, TileDMA-Dst).
2. **Data types restricted** to `UInt8`, `Int8`, `Float16`.  FP32 automatically down-converts.
3. **Strides must be multiples of 0x40 bytes** due to 64-byte cacheline / DMA alignment.
4. **Manual L2 cache**: 4 MiB scratchpad only for input/output. Programmer chooses residency via per-section flags.
5. **16-wide kernel DMA engine** underpins the “16 cores” marketing claim.  At 335 MHz a 32×32 MAC array yields ≈11 TOPS.
6. **Compiler flags** reveal internal allocators (`ffreuse`), cache strategies, and support for `h11` and `h13` target architectures (A11 & A13+ generations).
7. **Security**: Access gated by entitlement & AMFI; workarounds include live patching `amfid`, signing the interpreter, or kext patching.
8. **Execution model**: Userspace sends a 0x1000-word struct via `ANE_ProgramSendRequest` and waits on a Mach port.  Different client instances are required for compilation vs. execution.
9. **Activation & Fuseability**: 30+ activation modes, extensive op fusion (Conv+Bias+ReLU etc.) visible in compiler output.
10. **Reverse-engineering approach**: Combined Mach-O disassembly, LLDB live tracing (`ZinIrRegBitPrintOutDebug`), byte-wise diffing of HWX, and brute-force struct recovery.

---

## Conclusion

The `extra/accel/ane` tree documents a **full DIY tool-chain**: create CoreML graphs, convert to plists, compile plists to HWX, and finally execute HWX directly on ANE – completely outside Apple’s sanctioned APIs.  Beyond serving tinygrad, this corpus is a reference for:

- Understanding Apple’s custom NPU architecture.
- Learning how DMA-centric accelerators are programmed at the register level.
- Bypassing restrictive entitlements for research purposes.

With these notes future contributors can extend operator coverage, port the backend to newer SoCs (e.g. H14), or explore performance tuning by manipulating TD fields that Apple’s compiler never touches.



###############################################################################
###############################################################################


# Apple Neural Engine (ANE) / Apple NPU – Reverse-Engineering Notes

This document consolidates **all insights, take-aways, and technical revelations** extracted from **every single file** in `extra/accel/ane` inside the tinygrad repository.  It is intended both as learning material and as a map for anyone wishing to extend tinygrad’s experimental ANE back-end or to perform further research on Apple’s NPU.

---

## Directory Map

```
extra/accel/ane/
├── 1_build/           # Produce CoreML models
├── 2_compile/         # Transform models to ANE-executable HWX
├── 3_run/             # Execute HWX directly on the device
├── amfi/              # Entitlement / code-signing bypass helpers
├── lib/               # Stand-alone ANE loader + helpers (no tinygrad)
├── ops/               # Pre-compiled HWX kernels for inspection
├── tinygrad/          # tinygrad integration layer
├── README.md          # Polished, user-friendly overview
└── README.old         # Raw field notes while reversing
```

Below every file is listed, grouped by directory, followed by global findings.

---

## 1_build  — Creating Test Models

| File | Purpose | Key Learnings |
|------|---------|--------------|
| `.gitignore` | Ignore `DerivedData/` etc. | — |
| `coreml_ane.py` | Build a **64-element element-wise ADD** network with *coremltools* and run it, forcing execution on ANE. | Shows the *minimum viable* CoreML graph that triggers ANE; demonstrates that ANE accepts FP16 math when invoked through CoreML. |
| `run.swift` | Native Swift wrapper to load & run the created `.mlmodel`. | Confirms identical behaviour from Swift/Obj-C layer. |
| `test.mlmodel` | Generated model for manual inspection. | Mach-O package containing the CoreML spec. |

**Take-away:** *coremltools* is sufficient to feed ANE, but we still rely on Apple’s protected CoreML → Espresso → ANECompiler chain. The later directories show how to break free from that chain.

---

## 2_compile  — Turning Plists into HWX

### Top-level scripts

| File | Purpose / Functionality | Insights |
|------|------------------------|----------|
| `ane.py` | Pure-Python loader wrapping a private `libane.dylib`. Exposes `ANE.compile`, `ANE.run`, tensor helpers, register pack/unpack routines, and a pretty printer for ANE registers. | • **Operation layout** table (`ANE_Struct`) is the most accurate public description of an ANE TD (Tile Descriptor).<br>• Demonstrates that each op is exactly **0x300 bytes**.<br>• Confirms three data types: **UInt8 / Int8 / FP16**.<br>• Shows register bitfields by name, built from `aneregs.json`. |
| `aneregs.json` | 3-tuple `[byte, bit, width]` for **every single undocumented ANE register**. | Crucial for decoding TDs and for diffing compiler output. |
| `compile.mm` | Objective-C++ harness that calls the private **ANECompiler**’s symbol `ANECCompile`. Builds `model.hwx` from any `net.plist`. | Reveals the exact CoreFoundation dictionary keys the compiler expects (`InputNetworks`, `TargetArchitecture`, etc.). |
| `compile.m` | C wrapper used while experimenting with the same symbol. | Alternative stub. |
| `compile.sh` | Command-line convenience. | Documents required environment (codesign / entitlements). |
| `dcompile.py` | Dead-simple Python that spawns `compile.mm` repeatedly. | Automates batch experiments. |
| `hwx_parse.py` | Heavyweight diff/visualiser for two HWX binaries. Uses `macholib`, colorises differences, and maps bytes back to register names via `ANE.debug`. | Showed that **compiler emits relocation tables** for weights and IOSurfaces; helped reverse TD structure. |
| `struct_recover.py` | Script used to brute-force unknown struct offsets. | Assisted building `ANEREGS`. |

### Assets

Weights (`*.weights`) and sample net descriptors (`net.plist`) were captured during CoreML → Espresso compilation.  They allow offline reproduction of compiler behaviour.

`plists/` holds hand-written minimal plists for each op-code; `simple/` variants combine two or more ops to observe fused behaviour.

**Insights obtained from plists:**
- Mapping from high-level layer types to **Zin op-codes** (see README.old table).
- Observation that the compiler **fuses** kernel + activation + bias when possible.
- Confirmation that **Input/Output strides must be multiples of 0x40 bytes**.

---

## 3_run  — Talking to the Kernel Driver

| File | Purpose | Discoveries |
|------|---------|------------|
| `h11ane.h` | Re-created C++ header for **ANEServices** classes *H11ANEDevice* / *H11ANEDeviceController*. | Lists every user-client method name (power, logging, firmware, etc.).  Confirms two usage modes: `UsageCompile` (needs debugger) and `UsageWithProgram`. |
| `test.mm` | Full example that: 1) powers on ANE, 2) loads `model.hwx`, 3) allocates IOSurfaces, 4) sends `ANE_ProgramSendRequest`, 5) reads back result. | Shows the **Mach port IPC** pattern: async completion arrives on a receive port.  Also documents the unnamed 0x1000-word request struct. |
| `build.sh` | clang++ compile convenience. | Requires entitlements + `-framework IOSurface`. |
| `entitlements.xml` | Minimal XML granting `com.apple.ane.iokit-user-access`. | Must be injected via codesign or AMFI patch. |

---

## amfi  — Bypassing Code-Signing

| File | Description | Key Point |
|------|-------------|-----------|
| `new_patch.py` | LLDB script that **patches `/usr/libexec/amfid` in memory**, nop-ing the entitlement check. | Allows unsigned Python / clang binaries to access ANE without rebooting with `amfi_get_out_of_my_way`. |

---

## lib  — Standalone Loader (Not tinygrad-specific)

This sub-project appeared earlier in the research timeline and duplicates some functionality in `2_compile` + `3_run` but with a cleaner interface.

| File | Notes |
|------|------|
| `ane.mm` | Objective-C++ wrapper around `libane.dylib`; elegant C++ `Tensor` & `Program` classes. |
| `ane.py` | Pure python binding identical in spirit to `2_compile/ane.py` but without TD pretty-printer. |
| `aneregs.json` | Earlier version of the register map (superseded by the one in `2_compile`). |
| `build.sh` | Compiles `ane.mm` into `libane.dylib`. |
| `entitlements.xml` | Copy of minimal entitlement. |
| `h11ane.h` | Older, smaller extract of the C++ header. |
| `sign_python.sh` | Convenience codesign helper for the python interpreter itself. |
| `testconv.py` | End-to-end test: builds a 3×3 conv TD in python, uploads via `ane.py`, and verifies output. Shows manual TD creation without the compiler. |

---

## ops  — Pre-compiled Reference Kernels

Six `.hwx` blobs produced by Apple’s compiler.  Used to sanity-check the home-grown compiler and to diff against hand-written TDs.

| File | Op | Use |
|------|----|-----|
| `conv.hwx` | 3×3 FP16 convolution | Baseline for TD layout
| `relu.hwx` | Neuron only | Verified NE section
| `sigmoid.hwx` | Non-linear LUT | Verified custom LUT behaviour
| `concat.hwx`, `sum.hwx` | Multi-input TDs | Showed dual DMA source streams
| `gemm.hwx` | Large GEMM | Stress-tested row/plane strides

---

## tinygrad  — Integration Layer

| File | Purpose |
|------|---------|
| `ops_ane.py` | Implements tinygrad *LazyOp* → ANE translation for a subset of ops (conv, add, relu, gemm).  Picks between pre-compiled TDs in `ops/` or dynamically builds them via `2_compile/ane.py`. |

---

## README Files

| File | Nature | Highlights |
|------|--------|-----------|
| `README.md` | Clean public overview | Introduces high-level tensor layout, supported dtypes, manual L2 cache, 16-core figure, normal use-case flow, tips for extracting private frameworks. |
| `README.old` | **Raw research diary** | ‑ Full compiler backtraces<br>- Exact TD offset map (basis for `ANE_Struct`)<br>- List of every **Zin op-code** (0-48)<br>- List of **activation IDs** (0-31)<br>- Cache-hint enum (0-3)<br>- Compiler command-line printed by `aned` in dmesg.<br>- Notes on data-type coercions (FP32→FP16). |

---

## Global Technical Insights

1. **Tile Descriptor (TD) = 0x300 bytes** with seven logical sections (Header, KernelDMA, Common, TileDMA-Src, L2, NE, TileDMA-Dst).
2. **Data types restricted** to `UInt8`, `Int8`, `Float16`.  FP32 automatically down-converts.
3. **Strides must be multiples of 0x40 bytes** due to 64-byte cacheline / DMA alignment.
4. **Manual L2 cache**: 4 MiB scratchpad only for input/output. Programmer chooses residency via per-section flags.
5. **16-wide kernel DMA engine** underpins the “16 cores” marketing claim.  At 335 MHz a 32×32 MAC array yields ≈11 TOPS.
6. **Compiler flags** reveal internal allocators (`ffreuse`), cache strategies, and support for `h11` and `h13` target architectures (A11 & A13+ generations).
7. **Security**: Access gated by entitlement & AMFI; workarounds include live patching `amfid`, signing the interpreter, or kext patching.
8. **Execution model**: Userspace sends a 0x1000-word struct via `ANE_ProgramSendRequest` and waits on a Mach port.  Different client instances are required for compilation vs. execution.
9. **Activation & Fuseability**: 30+ activation modes, extensive op fusion (Conv+Bias+ReLU etc.) visible in compiler output.
10. **Reverse-engineering approach**: Combined Mach-O disassembly, LLDB live tracing (`ZinIrRegBitPrintOutDebug`), byte-wise diffing of HWX, and brute-force struct recovery.

---

## Conclusion

The `extra/accel/ane` tree documents a **full DIY tool-chain**: create CoreML graphs, convert to plists, compile plists to HWX, and finally execute HWX directly on ANE – completely outside Apple’s sanctioned APIs.  Beyond serving tinygrad, this corpus is a reference for:

- Understanding Apple’s custom NPU architecture.
- Learning how DMA-centric accelerators are programmed at the register level.
- Bypassing restrictive entitlements for research purposes.

With these notes future contributors can extend operator coverage, port the backend to newer SoCs (e.g. H14), or explore performance tuning by manipulating TD fields that Apple’s compiler never touches.



