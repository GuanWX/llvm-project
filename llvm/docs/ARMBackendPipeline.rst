======================================================
LLVM ARM Backend: From LLVM IR to Binary — A Detailed Pipeline Guide
======================================================

.. contents::
   :local:

.. note::

   This document provides a detailed walkthrough of how the LLVM ARM backend
   compiles LLVM IR into a binary (ELF object) file, with references to the
   corresponding source files and key methods.

Overview
========

The LLVM ARM backend transforms LLVM Intermediate Representation (IR) into ARM
machine code through a multi-stage pipeline. The overall flow can be summarized
as:

.. code-block:: text

   LLVM IR
     │
     ▼
   ┌─────────────────────────────┐
   │ 1. Target Initialization    │  ARMTargetMachine / ARMSubtarget
   └─────────────┬───────────────┘
                 │
                 ▼
   ┌─────────────────────────────┐
   │ 2. IR-Level Optimization    │  IR Passes (LoopStrengthReduce, etc.)
   └─────────────┬───────────────┘
                 │
                 ▼
   ┌─────────────────────────────┐
   │ 3. Instruction Selection    │  SelectionDAG / ARMISelLowering /
   │    (DAG → MachineInstr)     │  ARMISelDAGToDAG
   └─────────────┬───────────────┘
                 │
                 ▼
   ┌─────────────────────────────┐
   │ 4. Machine-Level Passes     │  Scheduling, Register Allocation,
   │    (MachineInstr → MachineInstr)  Prologue/Epilogue, Optimizations
   └─────────────┬───────────────┘
                 │
                 ▼
   ┌─────────────────────────────┐
   │ 5. Code Emission            │  AsmPrinter → MCInst →
   │    (MachineInstr → Binary)  │  MCCodeEmitter → AsmBackend →
   │                             │  ELFObjectWriter
   └─────────────────────────────┘
                 │
                 ▼
           ELF Object File (.o)


Stage 1: Target Initialization
==============================

Before code generation begins, LLVM must configure the target machine, select
the CPU features, and set up the pass pipeline.

ARMTargetMachine
----------------

**File:** ``llvm/lib/Target/ARM/ARMTargetMachine.cpp``

**Classes:** ``ARMBaseTargetMachine``, ``ARMLETargetMachine`` (little-endian),
``ARMBETargetMachine`` (big-endian)

The target machine is the central hub that holds the data layout, target triple,
and creates the pass pipeline for code generation.

Key Methods:

- ``createPassConfig(PassManagerBase &PM)`` — Creates the ``ARMPassConfig``
  object that defines the sequence of code-generation passes. This is the entry
  point for building the backend compilation pipeline.

- ``getSubtargetImpl(const Function &F)`` — Returns the ``ARMSubtarget`` for a
  given function. Different functions may have different subtargets if they use
  different target features (e.g., ``+neon``, ``+thumb2``).

- ``getTargetTransformInfo(const Function &F)`` — Provides cost-model
  information used by IR-level optimizations (e.g., vectorization, loop
  unrolling).

ARMSubtarget
------------

**File:** ``llvm/lib/Target/ARM/ARMSubtarget.cpp``

**Header:** ``llvm/lib/Target/ARM/ARMSubtarget.h``

**Class:** ``ARMSubtarget``

The subtarget encapsulates the CPU model, feature flags (NEON, VFP, MVE, Thumb,
etc.), and provides access to all target-specific helper objects.

Key Methods:

- ``ARMSubtarget(const Triple &TT, const std::string &CPU, const std::string &FS, ...)``
  — Constructor. Parses the CPU name and feature string, initializes the
  instruction info, register info, and frame lowering objects.

- ``initializeSubtargetDependencies(StringRef CPU, StringRef FS)`` — Called from
  the constructor. Resolves the CPU and feature string to set internal feature
  flags (e.g., ``HasNEON``, ``HasThumb2``, ``HasVFPv4``).

- ``getCallLowering()`` / ``getInstructionSelector()`` / ``getLegalizerInfo()``
  / ``getRegBankInfo()`` — Return GlobalISel components (an alternative
  instruction selection framework to SelectionDAG).

Top-Level Entry Point: ``addPassesToEmitFile``
----------------------------------------------

**File:** ``llvm/lib/CodeGen/CodeGenTargetMachineImpl.cpp``

**Method:** ``CodeGenTargetMachineImpl::addPassesToEmitFile(...)``

This is the top-level method that the driver calls to set up the full
compilation pipeline from LLVM IR to an output file (assembly, object, or
bitcode). It:

1. Creates the ``MachineModuleInfo``.
2. Calls ``addPassesToGenerateCode()`` to build the core code-generation
   pipeline (via ``ARMPassConfig``).
3. Adds the appropriate output streamer (assembly text or object binary).


Stage 2: IR-Level Optimization and Lowering
============================================

Before entering the code generator proper, several IR-level passes prepare the
LLVM IR for efficient code generation on ARM.

Pass Pipeline Configuration (ARMPassConfig)
-------------------------------------------

**File:** ``llvm/lib/Target/ARM/ARMTargetMachine.cpp``

**Class:** ``ARMPassConfig : public TargetPassConfig``

``ARMPassConfig`` configures the entire backend pass pipeline by overriding
``add*`` methods from ``TargetPassConfig``. The following methods define the
pass ordering:

.. list-table:: ARMPassConfig Pass Pipeline Methods
   :header-rows: 1
   :widths: 35 65

   * - Method
     - Description
   * - ``addIRPasses()``
     - Adds IR-level optimization passes (atomics expansion, merging of
       similar global constants, etc.)
   * - ``addCodeGenPrepare()``
     - Adds the CodeGenPrepare pass for IR-level lowering
   * - ``addPreISel()``
     - Adds passes that run before instruction selection (e.g.,
       ``ARMParallelDSP``, ``MVEGatherScatterLowering``,
       ``TypePromotionPass``)
   * - ``addInstSelector()``
     - Adds the instruction selector (``ARMISelDAGToDAG``)
   * - ``addPreRegAlloc()``
     - Adds passes before register allocation (e.g., ``A15SDOptimizer``,
       ``ARMOptimizeBarriersPass``)
   * - ``addPreSched2()``
     - Adds passes before the second scheduling pass (e.g.,
       ``ARMLowOverheadLoops``, ``Thumb2ITBlock``, ``ARMExpandPseudo``)
   * - ``addPreEmitPass()``
     - Adds passes before final code emission (e.g.,
       ``ARMConstantIslandPass``, ``ARMSLSHardening``)
   * - ``addPreEmitPass2()``
     - Adds late passes (e.g., ``ARMFixCortexA57AES1742098``,
       ``ARMBranchTargets``)

**GlobalISel alternative path:**

.. list-table:: GlobalISel Methods
   :header-rows: 1
   :widths: 35 65

   * - Method
     - Description
   * - ``addIRTranslator()``
     - Translates LLVM IR to Generic MIR (``IRTranslator``)
   * - ``addLegalizeMachineIR()``
     - Legalizes types and operations (``Legalizer``)
   * - ``addRegBankSelect()``
     - Assigns register banks (``RegBankSelect``)
   * - ``addGlobalInstructionSelect()``
     - Selects target instructions (``InstructionSelect``)


Stage 3: Instruction Selection (SelectionDAG)
==============================================

Instruction selection is the core phase that converts LLVM IR into target
machine instructions. LLVM uses a SelectionDAG-based approach as the default
for ARM.

Overview of SelectionDAG Instruction Selection
-----------------------------------------------

**File:** ``llvm/lib/CodeGen/SelectionDAG/SelectionDAGISel.cpp``

**Class:** ``SelectionDAGISel``

The base instruction selection framework works as follows:

1. **Build the DAG** — Each LLVM IR basic block is converted into a
   SelectionDAG, a directed acyclic graph of operations.
2. **Legalize** — Illegal types and operations are legalized (e.g., 64-bit
   operations on a 32-bit CPU).
3. **Optimize** — DAG-level optimizations (constant folding, CSE, etc.).
4. **Select** — Pattern-match DAG nodes to target instructions.
5. **Schedule** — Schedule the selected instructions.
6. **Emit** — Emit ``MachineInstr`` instructions into ``MachineBasicBlock``.

Key Methods:

- ``runOnMachineFunction(MachineFunction &MF)`` — Entry point. Initializes
  analyses and iterates over all basic blocks.

- ``SelectAllBasicBlocks(const Function &Fn)`` — Iterates through all basic
  blocks and calls ``SelectBasicBlock()`` for each.

- ``CodeGenAndEmitDAG()`` — Performs legalization, optimization, scheduling,
  and instruction emission for one DAG.

ARMISelLowering (Legalization & Custom Lowering)
-------------------------------------------------

**File:** ``llvm/lib/Target/ARM/ARMISelLowering.cpp``

**Header:** ``llvm/lib/Target/ARM/ARMISelLowering.h``

**Class:** ``ARMTargetLowering : public TargetLowering``

This class tells the legalizer which operations are supported natively on ARM,
which need to be expanded, and which require custom lowering. It is one of the
largest files in the ARM backend (~21,000+ lines).

Key Methods:

- **Constructor** ``ARMTargetLowering(const TargetMachine &TM, const ARMSubtarget &STI)``
  — Registers the legality of each operation and type. For example, it marks
  ``ISD::SDIV`` as ``Expand`` (not natively supported on all ARM cores) and
  ``ISD::BSWAP`` as ``Expand`` for Thumb1.

- ``LowerOperation(SDValue Op, SelectionDAG &DAG)`` — Main dispatcher. Routes
  each operation that requires custom lowering to the appropriate handler. For
  example:

  - ``ISD::GlobalAddress`` → ``LowerGlobalAddressELF()`` / ``LowerGlobalAddressDarwin()``
  - ``ISD::BR_CC`` → ``LowerBR_CC()``
  - ``ISD::SELECT_CC`` → ``LowerSELECT_CC()``
  - ``ISD::RETURNADDR`` → ``LowerRETURNADDR()``

- ``LowerFormalArguments(...)`` — Lowers function argument handling according
  to the ARM calling convention (AAPCS, AAPCS-VFP). Maps arguments to registers
  (R0–R3, S0–S15, D0–D7) or the stack.

- ``LowerCall(...)`` — Lowers function calls. Marshals arguments into the
  correct registers/stack locations, emits the call instruction (BL, BLX), and
  extracts return values.

- ``LowerReturn(...)`` — Lowers function return. Moves return values into the
  return registers (R0–R1, S0, D0) and emits the return instruction.

- ``ReplaceNodeResults(SDNode *N, ...)`` — Handles cases where a node produces
  illegal result types and provides custom replacement.

ARMISelDAGToDAG (Pattern Matching)
-----------------------------------

**File:** ``llvm/lib/Target/ARM/ARMISelDAGToDAG.cpp``

**Class:** ``ARMDAGToDAGISel : public SelectionDAGISel``

This class performs the final instruction selection, converting legalized
SelectionDAG nodes into ARM machine instructions using pattern matching.

Key Methods:

- ``Select(SDNode *N)`` — The core method. For each DAG node, it attempts to
  match a pattern and emit the corresponding ARM machine instruction. It
  handles special cases (e.g., ``ISD::Constant``, ``ISD::FrameIndex``,
  ``ARMISD::*`` nodes) and falls back to TableGen-generated patterns for
  standard cases via ``SelectCode(N)``.

- ``runOnMachineFunction(MachineFunction &MF)`` — Caches the subtarget
  reference and delegates to the parent class.

- ``PreprocessISelDAG()`` — Pre-processes the DAG before selection. Performs
  ARM-specific DAG optimizations.

- ``SelectRegShifterOperand(...)`` / ``SelectImmShifterOperand(...)`` —
  Complex pattern matchers for ARM's flexible barrel-shifter operands
  (e.g., ``LSL #2``, ``ROR R3``).

- ``SelectAddrModeImm12(SDValue N, SDValue &Base, SDValue &OffImm)`` — Matches
  addressing modes for load/store instructions.

TableGen-Generated Instruction Descriptions
--------------------------------------------

**Files:**

- ``llvm/lib/Target/ARM/ARM.td`` — Top-level TableGen description
- ``llvm/lib/Target/ARM/ARMInstrInfo.td`` — ARM instruction definitions
- ``llvm/lib/Target/ARM/ARMInstrThumb.td`` — Thumb instruction definitions
- ``llvm/lib/Target/ARM/ARMInstrThumb2.td`` — Thumb-2 instruction definitions
- ``llvm/lib/Target/ARM/ARMInstrNEON.td`` — NEON SIMD instruction definitions
- ``llvm/lib/Target/ARM/ARMInstrVFP.td`` — VFP floating-point instructions
- ``llvm/lib/Target/ARM/ARMInstrMVE.td`` — M-profile Vector Extension (MVE)

These ``.td`` files define the instruction encoding, operands, assembly syntax,
and pattern-matching rules. The ``llvm-tblgen`` tool processes them to generate
C++ code for instruction selection, encoding, and printing. In particular, the
``SelectCode(N)`` method called from ``ARMDAGToDAGISel::Select()`` uses the
patterns defined in these ``.td`` files to match SelectionDAG nodes to ARM
machine instructions.


Stage 4: Machine-Level Passes
==============================

After instruction selection, the code is in the form of ``MachineInstr``
objects within ``MachineFunction`` and ``MachineBasicBlock`` containers.
Several optimization and transformation passes operate at this level.

Machine Instruction Scheduling
-------------------------------

**File:** ``llvm/lib/CodeGen/MachineScheduler.cpp``

The instruction scheduler reorders ``MachineInstr`` instructions to improve
performance by reducing pipeline stalls and maximizing instruction-level
parallelism.

- ``ScheduleDAGMILive::schedule()`` — Pre-register-allocation scheduler.
- ``PostMachineScheduler::runOnMachineFunction(...)`` — Post-register-allocation
  scheduler.

ARM uses ``createARMPostMachineScheduler()`` (defined in
``ARMTargetMachine.cpp``) which returns a custom scheduling strategy.

Register Allocation
-------------------

**Files:**

- ``llvm/lib/Target/ARM/ARMBaseRegisterInfo.cpp`` — ARM register information
- ``llvm/lib/CodeGen/RegAllocGreedy.cpp`` — Greedy register allocator (default)

**Class:** ``ARMBaseRegisterInfo : public ARMGenRegisterInfo``

This class provides ARM-specific register allocation information.

Key Methods:

- ``getCalleeSavedRegs(const MachineFunction *MF)`` — Returns the list of
  callee-saved registers (R4–R11, LR on ARM).

- ``getReservedRegs(const MachineFunction &MF)`` — Returns the set of registers
  that cannot be allocated (SP, PC, and optionally FP).

- ``getRegAllocationHints(Register VirtReg, ...)`` — Provides hints to the
  register allocator for better allocation (e.g., for paired registers in
  ``LDRD``/``STRD`` instructions).

- ``eliminateFrameIndex(MachineBasicBlock::iterator II, int SPAdj, ...)`` —
  Replaces frame index operands with concrete ``SP+offset`` or ``FP+offset``
  addressing after register allocation.

Prologue/Epilogue Insertion (Frame Lowering)
---------------------------------------------

**File:** ``llvm/lib/Target/ARM/ARMFrameLowering.cpp``

**Class:** ``ARMFrameLowering : public TargetFrameLowering``

After register allocation, the compiler inserts function prologue and epilogue
code for stack frame setup/teardown.

Key Methods:

- ``emitPrologue(MachineFunction &MF, MachineBasicBlock &MBB)`` — Emits the
  function prologue: pushes callee-saved registers (``PUSH {R4-R11, LR}``),
  adjusts the stack pointer (``SUB SP, SP, #framesize``), and sets up the
  frame pointer if needed.

- ``emitEpilogue(MachineFunction &MF, MachineBasicBlock &MBB)`` — Emits the
  function epilogue: restores the stack pointer, pops callee-saved registers
  (``POP {R4-R11, PC}``), and returns.

- ``spillCalleeSavedRegisters(...)`` — Spills callee-saved registers to the
  stack at function entry.

- ``restoreCalleeSavedRegisters(...)`` — Restores callee-saved registers from
  the stack at function exit.

- ``determineCalleeSaves(MachineFunction &MF, BitVector &SavedRegs, ...)`` —
  Determines which registers must be saved by analyzing register usage across
  the function.

There are also variant frame lowering classes for different ARM modes:

- ``Thumb1FrameLowering`` (``llvm/lib/Target/ARM/Thumb1FrameLowering.cpp``)
- ``ThumbRegisterInfo`` (for Thumb-mode-specific register handling)

ARM-Specific Machine Passes
----------------------------

Several ARM-specific passes optimize the machine code:

.. list-table:: ARM-Specific Machine Passes
   :header-rows: 1
   :widths: 30 30 40

   * - Pass
     - File
     - Description
   * - ``ARMConstantIslandPass``
     - ``ARMConstantIslandPass.cpp``
     - Scatters constant pool entries to keep them within PC-relative addressing
       range (ARM has a limited 12-bit offset for LDR from constant pool).
   * - ``ARMExpandPseudoPass``
     - ``ARMExpandPseudoInsts.cpp``
     - Expands pseudo-instructions (e.g., ``MOVi32imm`` → ``MOVW`` + ``MOVT``)
       into real ARM instructions.
   * - ``Thumb2ITBlockPass``
     - ``Thumb2ITBlockPass.cpp``
     - Inserts IT (If-Then) blocks for conditional execution in Thumb-2 mode.
   * - ``ARMLowOverheadLoopsPass``
     - ``ARMLowOverheadLoops.cpp``
     - Optimizes low-overhead loops using ARM's WLS/DLS/LE loop instructions
       (for M-profile with MVE).
   * - ``ARMOptimizeBarriersPass``
     - ``ARMOptimizeBarriersPass.cpp``
     - Removes redundant memory barrier instructions (DMB).
   * - ``A15SDOptimizerPass``
     - ``A15SDOptimizer.cpp``
     - Optimizes S-register to D-register forwarding for Cortex-A15.
   * - ``ARMSLSHardeningPass``
     - ``ARMSLSHardening.cpp``
     - Inserts speculative load hardening for Spectre mitigation.
   * - ``ARMBranchTargetsPass``
     - ``ARMBranchTargets.cpp``
     - Inserts BTI (Branch Target Identification) instructions for pointer
       authentication (ARMv8.1-M).


Stage 5: Code Emission (MachineInstr → Binary)
===============================================

The final stage converts ``MachineInstr`` objects to actual binary machine code.
This involves several layers of abstraction in the MC (Machine Code) layer.

Emission Pipeline Overview
--------------------------

.. code-block:: text

   MachineInstr  (per-function machine instructions)
       │
       ▼
   ARMAsmPrinter           ── Converts MachineInstr → MCInst
       │
       ▼
   MCStreamer               ── Streams MCInst to output
       │
       ├── MCObjectStreamer  (for .o object files)
       │       │
       │       ▼
       │   ARMMCCodeEmitter  ── Encodes MCInst → bytes
       │       │
       │       ▼
       │   ARMAsmBackend     ── Applies fixups / relaxation
       │       │
       │       ▼
       │   ARMELFObjectWriter ── Writes ELF sections & relocations
       │
       └── MCAsmStreamer     (for .s assembly text output)

ARMAsmPrinter
-------------

**File:** ``llvm/lib/Target/ARM/ARMAsmPrinter.cpp``

**Class:** ``ARMAsmPrinter : public AsmPrinter``

Converts ``MachineInstr`` (the compiler's internal machine instruction
representation) into ``MCInst`` (the lightweight MC-layer instruction
representation for encoding/printing).

Key Methods:

- ``runOnMachineFunction(MachineFunction &MF)`` — Entry point for each
  function. Sets up state and calls the base class to iterate over all
  instructions.

- ``emitInstruction(const MachineInstr *MI)`` — Converts a single
  ``MachineInstr`` into one or more ``MCInst`` objects and sends them to the
  ``MCStreamer``. Handles pseudo-instructions, inline assembly, and
  ARM-specific special cases.

- ``emitFunctionEntryLabel()`` — Emits the function's entry label, including
  Thumb/ARM mode markers (``$a`` / ``$t`` mapping symbols).

- ``emitStartOfAsmFile(Module &M)`` — Emits file-level directives such as
  ``.syntax unified``, ``.eabi_attribute``, and CPU/FPU feature attributes.

- ``emitEndOfAsmFile(Module &M)`` — Emits end-of-file content, including
  attribute sections for ELF objects.

ARMMCCodeEmitter
----------------

**File:** ``llvm/lib/Target/ARM/MCTargetDesc/ARMMCCodeEmitter.cpp``

**Class:** ``ARMMCCodeEmitter : public MCCodeEmitter``

Encodes ``MCInst`` instructions into their binary representation (the actual
bytes of machine code).

Key Methods:

- ``encodeInstruction(const MCInst &MI, SmallVectorImpl<char> &CB, ...)`` —
  Main entry point. Encodes an entire instruction to bytes. Handles both ARM
  (32-bit) and Thumb (16/32-bit) encoding formats.

- ``getBinaryCodeForInstr(const MCInst &MI, ...)`` — TableGen-generated method
  that returns the binary encoding for a given instruction. Calls various
  ``get*OpValue()`` methods for operand encoding.

- ``getMachineOpValue(const MCInst &MI, const MCOperand &MO, ...)`` — Returns
  the binary encoding for a single operand. For register operands, returns the
  register encoding. For immediate operands, returns the immediate value. For
  expression operands, creates a fixup for later resolution.

- ``getAddrModeImm12OpValue(...)`` / ``getAddrMode5OpValue(...)`` — Encodes
  ARM-specific addressing modes (base+offset, etc.).

ARMAsmBackend
-------------

**File:** ``llvm/lib/Target/ARM/MCTargetDesc/ARMAsmBackend.cpp``

**Class:** ``ARMAsmBackend : public MCAsmBackend``

Handles instruction relaxation (widening narrow instructions when they can't
reach their target) and applies fixups to the encoded instruction bytes.

Key Methods:

- ``applyFixup(const MCAssembler &Asm, const MCFixup &Fixup, ...)`` — Applies
  a fixup to the instruction bytes. For example, resolves a branch target
  offset and patches it into the instruction encoding.

- ``relaxInstruction(MCInst &Inst, const MCSubtargetInfo &STI)`` — Relaxes a
  narrow instruction to a wider encoding. For example, relaxes a Thumb
  ``tB`` (short branch) to ``t2B`` (wide branch) when the branch target is too
  far away.

- ``writeNopData(raw_ostream &OS, uint64_t Count, ...)`` — Writes NOP
  instructions for alignment padding.

- ``getFixupKindInfo(MCFixupKind Kind)`` — Returns metadata (offset, bit size,
  flags) for each ARM fixup kind.

ARMELFObjectWriter
------------------

**File:** ``llvm/lib/Target/ARM/MCTargetDesc/ARMELFObjectWriter.cpp``

**Class:** ``ARMELFObjectWriter : public MCELFObjectTargetWriter``

Writes the final ELF object file, including translating internal fixups into
ELF relocations.

Key Methods:

- ``getRelocType(MCContext &Ctx, const MCValue &Target, const MCFixup &Fixup, bool IsPCRel)``
  — Converts an LLVM fixup into the appropriate ELF relocation type
  (e.g., ``R_ARM_CALL``, ``R_ARM_THM_JUMP24``, ``R_ARM_ABS32``,
  ``R_ARM_MOVW_ABS_NC``).

- ``needsRelocateWithSymbol(const MCValue &, const MCSymbol &, unsigned Type)``
  — Determines whether a relocation must reference a symbol (rather than a
  section). Some relocations (like ``R_ARM_GOT32``) always need symbol
  references.

MCTargetDesc Registration
-------------------------

**File:** ``llvm/lib/Target/ARM/MCTargetDesc/ARMMCTargetDesc.cpp``

This file registers all the MC-layer components so that the LLVM infrastructure
can create them on demand.

Key Factory Functions:

- ``createARMMCInstrInfo()`` — Creates the instruction description table.
- ``createARMMCRegisterInfo(const Triple &)`` — Creates the register
  description table.
- ``createARMMCAsmBackend(...)`` — Creates the assembly backend.
- ``createARMMCCodeEmitter(...)`` — Creates the code emitter.
- ``createARMELFObjectWriter(...)`` — Creates the ELF object writer.


Additional Components
=====================

Assembly Parser
---------------

**File:** ``llvm/lib/Target/ARM/AsmParser/ARMAsmParser.cpp``

**Class:** ``ARMAsmParser : public MCTargetAsmParser``

Parses ARM assembly text (``.s`` files) into ``MCInst`` objects. This is
used by the integrated assembler (``llvm-mc``) and when handling inline
assembly in C/C++ code.

Key Methods:

- ``parseInstruction(ParseInstructionInfo &Info, StringRef Name, SMLoc NameLoc, OperandVector &Operands)``
  — Parses one assembly instruction (mnemonic + operands).

- ``ParseDirective(AsmToken DirectiveID)`` — Handles ARM-specific directives
  such as ``.syntax``, ``.thumb``, ``.arm``, ``.req``, ``.unreq``,
  ``.object_arch``.

- ``MatchAndEmitInstruction(...)`` — Validates parsed operands against the
  instruction definitions and emits the final ``MCInst``.

Disassembler
------------

**File:** ``llvm/lib/Target/ARM/Disassembler/ARMDisassembler.cpp``

**Class:** ``ARMDisassembler : public MCDisassembler``

Decodes binary ARM machine code back into ``MCInst`` objects (used by
``llvm-objdump`` and other disassembly tools).

Key Method:

- ``getInstruction(MCInst &MI, uint64_t &Size, ArrayRef<uint8_t> Bytes, uint64_t Address, raw_ostream &CStream)``
  — Decodes one instruction from a byte array.


Complete File Reference
========================

Below is a quick-reference table mapping each pipeline stage to its primary
source files and classes:

.. list-table:: Pipeline Stage to Source File Mapping
   :header-rows: 1
   :widths: 20 35 25 20

   * - Stage
     - Source File
     - Class
     - Key Method
   * - Target Init
     - ``ARMTargetMachine.cpp``
     - ``ARMBaseTargetMachine``
     - ``createPassConfig()``
   * - Target Init
     - ``ARMSubtarget.cpp``
     - ``ARMSubtarget``
     - constructor
   * - Pass Pipeline
     - ``ARMTargetMachine.cpp``
     - ``ARMPassConfig``
     - ``addInstSelector()``
   * - Top-Level Entry
     - ``CodeGenTargetMachineImpl.cpp``
     - ``CodeGenTargetMachineImpl``
     - ``addPassesToEmitFile()``
   * - IR Lowering
     - ``ARMISelLowering.cpp``
     - ``ARMTargetLowering``
     - ``LowerOperation()``
   * - DAG ISel
     - ``ARMISelDAGToDAG.cpp``
     - ``ARMDAGToDAGISel``
     - ``Select()``
   * - DAG ISel Framework
     - ``SelectionDAGISel.cpp``
     - ``SelectionDAGISel``
     - ``CodeGenAndEmitDAG()``
   * - Register Info
     - ``ARMBaseRegisterInfo.cpp``
     - ``ARMBaseRegisterInfo``
     - ``getCalleeSavedRegs()``
   * - Frame Lowering
     - ``ARMFrameLowering.cpp``
     - ``ARMFrameLowering``
     - ``emitPrologue()``
   * - Constant Islands
     - ``ARMConstantIslandPass.cpp``
     - ``ARMConstantIslands``
     - ``runOnMachineFunction()``
   * - Pseudo Expand
     - ``ARMExpandPseudoInsts.cpp``
     - ``ARMExpandPseudo``
     - ``ExpandMI()``
   * - Asm Printing
     - ``ARMAsmPrinter.cpp``
     - ``ARMAsmPrinter``
     - ``emitInstruction()``
   * - Code Encoding
     - ``ARMMCCodeEmitter.cpp``
     - ``ARMMCCodeEmitter``
     - ``encodeInstruction()``
   * - Fixup/Relaxation
     - ``ARMAsmBackend.cpp``
     - ``ARMAsmBackend``
     - ``applyFixup()``
   * - ELF Writing
     - ``ARMELFObjectWriter.cpp``
     - ``ARMELFObjectWriter``
     - ``getRelocType()``
   * - MC Registration
     - ``ARMMCTargetDesc.cpp``
     - (free functions)
     - ``createARMMCCodeEmitter()``
   * - Asm Parsing
     - ``ARMAsmParser.cpp``
     - ``ARMAsmParser``
     - ``parseInstruction()``
   * - Disassembly
     - ``ARMDisassembler.cpp``
     - ``ARMDisassembler``
     - ``getInstruction()``

All ARM backend source files are located under ``llvm/lib/Target/ARM/`` and
its subdirectories (``MCTargetDesc/``, ``AsmParser/``, ``Disassembler/``,
``TargetInfo/``, ``Utils/``).

The TableGen instruction descriptions are in ``*.td`` files in the same
directory, with ``ARM.td`` as the top-level file that includes all others.
