# Instruction Address Misaligned Exception Semantics

This proposal standardizes the handling of instruction-address-misaligned exceptions. The specification establishes the required effect when the underlying ISA would raise an instruction-address-misaligned exception, enabling consistent proof verification behavior and stronger correctness and security guarantees.

## Motivation

* **Consistent program behavior**
  Developers targeting multiple zkVM must be able to rely on consistent, predictable behavior when their program encounters a fault. Without a standard, the effect of an instruction-address-misaligned exception varies across implementations.

* **Proof system integrity**
  Without a standard, the effect of such a fault on the proof is undefined, which risks allowing a valid proof to be produced for an execution that did not reach a valid completion state.

* **Developer experience**
  Explicit and consistent fault semantics simplify portability and debugging across zkVM backends.

## Relationship to Other Standards

This standard applies when the underlying ISA raises an instruction-address-misaligned exception. For RISC-V targets with IALIGN=32, a taken branch or jump whose effective target is not 4-byte aligned raises this exception. For the `riscv64gc-unknown-none-elf` target defined in the [RISC-V target standard](../riscv-target/target.md), the C extension sets IALIGN=16, so standard branch and jump instructions cannot raise this exception: their targets are 2-byte aligned, and `JALR` clears bit 0 of its computed target. The termination requirement below applies to targets on which the exception can occur.

The required effect of an instruction-address-misaligned exception is abnormal termination as defined by the execution termination semantics standard.

Note that the `Zicclsm` extension, which mandates transparent handling of misaligned *data* memory accesses, has no bearing on instruction fetch alignment. `Zicclsm` applies exclusively to load and store operations. A zkVM implementing `Zicclsm` must still apply this standard when an instruction-address-misaligned exception would be raised.

## Specification

### Instruction-Address-Misaligned Exception

When the underlying ISA would raise an instruction-address-misaligned exception, the zkVM must trigger abnormal termination as defined by the execution termination semantics standard.

No recovery or continuation strategy — including but not limited to rounding the target address down to the nearest aligned address — is permitted.

### Rationale

A misaligned instruction address is not a recoverable or expected condition in any well-functioning program. Regardless of whether a zkVM exposes an exception handling mechanism, the observable behavior of the program must be consistent across implementations: the program must terminate abnormally.

### Verification

Verification behavior follows directly from the execution termination semantics standard. For a Type 1 verifier, verification fails for any proof of an execution that terminated due to an instruction-address-misaligned exception. For a Type 2 verifier, the exception may translate to a non-zero exit code, and verification can succeed only if the verifier explicitly accepts that exit code.

