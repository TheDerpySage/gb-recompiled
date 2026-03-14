# Recompiler Correctness Audit Plan

## Goal

Find and eliminate semantic mismatches between:

1. The decoded SM83 instruction stream
2. The IR/lowering pipeline
3. The generated C output
4. The runtime/interpreter behavior

The recent `(HL)` ALU bug in Pokemon Blue and Tetris was not an isolated audio issue. It exposed a broader problem: the project currently proves *coverage* much better than it proves *correctness*.

## What The Audit Found

### 1. The current "ground truth" workflow only checks coverage

`tools/compare_ground_truth.py` verifies that traced instruction addresses appear in generated C comments. It does **not** compare:

- register state
- flags
- PC/SP behavior
- bank state
- memory writes
- timing side effects

That is why a bug like:

- decoded instruction: `AND (HL)`
- generated code: `gb_and8(ctx, ctx->b);`

could pass the existing workflow.

### 2. Operand handling is still structurally fragile

The active emitter path in `recompiler/src/codegen/c_emitter.cpp` still relies on raw register indices plus special cases like:

- `6 == (HL)` for 8-bit operands
- `4 == AF` for 16-bit stack operations

The current fix patched the ALU family, but the design is still easy to break because:

- `recompiler/src/ir/ir_builder.cpp` uses magic values like `Operand::reg8(6)`
- the emitter still has multiple generic `reg8_names[...]` sites
- special operand behavior is encoded by convention instead of type

This is the same bug class that caused the Pokemon Blue audio failure.

### 3. There is stale duplicate code in the core pipeline

The active generator path is `generate_output()` in `recompiler/src/codegen/c_emitter.cpp`.

But the same file also contains a large older `CEmitter` class with obsolete semantics and obsolete identifiers, including references like:

- `ctx->A`
- `FLAG_Z(ctx)`
- `ctx->ime_scheduled`
- many `gbrt_*` helpers that are no longer the canonical runtime API

Likewise, `recompiler/src/ir/ir_builder.cpp` still declares and defines unused lowering helpers:

- `lower_jump`
- `lower_call`
- `lower_ret`
- `lower_io`

This is dangerous because it creates multiple versions of "how instruction X works", and only one of them is actually live.

### 4. The generated timing model is intentionally approximate

The active emitter batches `gb_tick()` at basic-block boundaries instead of ticking per instruction.

This matters because `gb_tick()` drives:

- timer/DIV updates
- interrupt synchronization
- DMA progress
- PPU sync
- APU stepping
- `ime_pending -> ime` transition

Concrete example:

- `EI` emits `ctx->ime_pending = 1`
- `ime_pending` is only applied in `gb_tick()` in `runtime/src/gbrt.c`
- if `gb_tick()` is delayed until the end of a basic block, IME may become active after several instructions instead of after exactly one

So the current codegen can be functionally wrong even when every opcode lowers to the "right" helper call.

### 5. Control-flow analysis loses some bank/context precision

`recompiler/include/recompiler/analyzer.h` stores basic block successors as `std::vector<uint16_t>`, which drops bank information.

Later, `recompiler/src/analyzer.cpp` reconstructs successor addresses using the current block's bank. That is acceptable for many same-bank edges, but it is not a sound representation for cross-bank control flow discovered earlier in analysis.

This is more of a CFG quality problem than the immediate `(HL)` bug, but it increases the chance of:

- wrong function grouping
- missed compiled targets
- overuse of dispatcher/interpreter fallback
- hidden bank-sensitive bugs

### 6. Debugging counters and trace behavior are not fully trustworthy

Instruction counting currently happens in multiple places:

- generated dispatch loop
- interpreter loop
- `gb_step()`
- `gb_tick()`

That makes instruction limits and some trace-driven debugging less reliable than they appear, especially when comparing interpreter and generated execution.

## Root Causes

1. No differential semantic validation against the interpreter
2. Operand semantics encoded with magic indices instead of operand kinds
3. Stale duplicate implementations left beside the live code path
4. Timing accuracy traded away in codegen without validation barriers
5. Audit tooling focused on discovery/coverage instead of correctness

## Fix Plan

### Phase 1: Build a semantic oracle

Priority: highest

Add a differential test harness that runs:

- the interpreter
- the generated dispatch path

from the same ROM and initial CPU state for a bounded instruction window, then compares after each instruction:

- `pc`, `sp`
- `af`, `bc`, `de`, `hl`
- unpacked flags
- `ime`, `ime_pending`, `halted`, `stopped`, `halt_bug`
- `rom_bank`, `ram_bank`, `mbc_mode`, other banking state as needed
- selected IO/timer state at minimum
- optionally memory writes via a write log

This harness should be able to stop on the first divergence and print:

- bank:pc
- decoded instruction
- generated function/block if available
- before/after state for both paths

Deliverables:

- a new local differential tool under `tools/`
- a small deterministic ROM set for repeatable checks
- at least one headless regression case for Pokemon Blue and Tetris

### Phase 2: Remove stale code paths and collapse to one canonical lowering/emission path

Priority: highest

Clean up the pipeline so there is exactly one live implementation for each stage.

Required work:

- remove or quarantine the unused legacy `CEmitter` path
- remove or rewire unused `IRBuilder` helper methods
- remove dead code that references obsolete runtime APIs
- make the active lowering/emission path obvious from the source tree

Success criteria:

- no parallel "old" and "new" emitter logic in the same file
- no unused lowering helpers that encode instruction semantics differently

### Phase 3: Replace magic operand indices with explicit operand kinds

Priority: highest

Refactor operand handling so `(HL)` and similar special cases are represented as typed operands, not integer conventions.

Recommended direction:

- stop encoding `(HL)` as `Operand::reg8(6)`
- use `OperandType::MEM_REG16` or a dedicated operand/helper for indirect 8-bit operands
- centralize emission helpers for:
  - reg8 register read
  - reg8 register write
  - `(HL)` read
  - `(HL)` write
  - AF push/pop masking

Add static audits that fail generation if the active emitter tries to stringify:

- `reg8 == 6` through the generic register-name path
- `AF` through a generic reg16 path where masking is required

### Phase 4: Restore instruction-accurate timing first, then optimize safely

Priority: highest

The current basic-block cycle batching is too risky to keep as the default while correctness is still being established.

Short-term:

- switch generated code back to per-instruction `gb_tick()`
- keep branch/call/return timing exact
- preserve `EI`, `DI`, `HALT`, `STOP`, interrupt, timer, DMA, PPU, and APU ordering relative to instruction execution

Only after differential tests are green should we reintroduce any batching, and then only behind proofs/tests showing the batched region is timing-safe.

A practical compromise is:

- correctness mode: always per-instruction
- optimized mode: optional safe coalescing for blocks proven not to touch timing-sensitive state

### Phase 5: Harden bank-aware control-flow analysis

Priority: medium

Refactor analysis structures so CFG edges preserve bank context explicitly.

Required work:

- store successors as full `(bank, addr)` values
- propagate bank-aware edges through function grouping
- add focused tests for:
  - banked `JP nn`
  - banked `CALL nn`
  - `JP HL`
  - RST 28 jump table patterns
  - short thunk functions

This will not fix the recent bug directly, but it will reduce silent fallback and analysis drift.

### Phase 6: Add regression gates for known fragile classes

Priority: medium

Add targeted checks for the bug classes already proven risky:

- `(HL)` ALU read operands
- `(HL)` read-modify-write instructions
- AF push/pop masking
- EI/DI/interrupt delay behavior
- HALT bug path
- banked jump/call targets

Recommended layers:

- unit tests for decoder/lowering metadata
- emitter golden tests for representative IR instructions
- differential execution smoke tests for small ROM snippets
- representative ROM smoke suite

### Phase 7: Fix debug/accounting drift

Priority: low, but worth doing early if it blocks validation

Clean up:

- instruction counting
- trace entry logging
- debug-only counters that currently overcount or disagree between paths

The semantic harness from Phase 1 needs trustworthy step accounting.

## Recommended Execution Order

1. Differential semantic harness
2. Remove stale emitter/lowering paths
3. Refactor operand representation away from magic indices
4. Re-enable per-instruction timing in generated code
5. Harden bank-aware CFG
6. Add permanent regression tests

## Immediate Next Slice

The next implementation slice should be:

1. Add a differential interpreter-vs-generated harness
2. Disable block-level tick batching in generated code
3. Add a small static audit that rejects active emitter sites which use generic reg8 formatting for `(HL)`

That combination will catch the next bug class faster than continuing to fix ROM-specific symptoms one at a time.
