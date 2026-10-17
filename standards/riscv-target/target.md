# Standardizing a RISC-V target for Ethereum

This proposal standardizes a RISC-V target for zkEVMs used on Ethereum, choosing an ISA and ABI supported by stock toolchains.

Proposed target:

```
riscv64gc-unknown-none-elf
```

with the additional platform guarantee of the `Zicclsm` extension.

## Motivation

Different zkEVMs currently target different RISC-V triples, standardization helps to normalize benchmarks across different zkVMs with respects to the circuit being proven, focus formal verification efforts and reduce the long term complexity of auditing the circuits.

RV64GC with LP64D is a common baseline in the 64-bit RISC-V software ecosystem. Choosing it allows guests to use existing compiler targets without maintaining a custom backend, and keeps guest builds on compiler configurations that are shared with the wider RISC-V ecosystem.

## Goals

- Define a RISC-V instruction set for zkEVMs that want to prove Ethereum blocks.
- Use an ISA and ABI that stock compilers support without custom targets.

## Non-Goals

- Define a RISC-V instruction set for proving arbitrary general purpose programs
- Not support every language directly
- We are not defining a standard for other targets and the standardization of this target has no precedence on other targets. For example, one could envision WASM-WASI being a suitable target for WASM.
- Mandate how a zkVM implements an instruction (dedicated circuit, transpilation, software emulation). Only the architectural behavior is mandated.


| **Category**                  | **Proposed Setting**                                         |
| ----------------------------- | ------------------------------------------------------------ |
| **ISA Base**                  | `RV64I`                                                      |
| **Extensions**                | `M`, `A`, `F`, `D`, `C`, `Zicsr`, `Zifencei`, `Zicclsm`      |
| **Compressed (`C`)**          | *Required*                                                   |
| **Floating Point (`F`, `D`)** | *Required* (IEEE 754-2008, bit-exact)                        |
| **Privileged Mode**           | Machine (`M`) only                                           |
| **Syscalls / Environment**    | None                                                         |
| **ABI**                       | `LP64D` (hard-float)                                         |
| **Object Format**             | ELF, statically linked                                       |
| **Endianness**                | Little-endian                                                |
| **Memory Model**              | Flat, no MMU, no paging                                      |

`G` is shorthand for `IMAFD_Zicsr_Zifencei`, so the ISA string is `RV64IMAFDC_Zicsr_Zifencei_Zicclsm`. The mandatory extensions overlap with the ratified [RVA20U64](https://github.com/riscv/riscv-profiles/blob/main/src/profiles.adoc#512-rva20u64-mandatory-extensions) profile, including its misaligned-access guarantee. This standard does not claim RVA20U64 conformance: for example, `Zicntr` is not required here.

## Rationale

### Default compilers

Stock toolchains support this ISA and ABI, although OS and runtime requirements remain separate:

- **Rust:** `riscv64gc-unknown-none-elf` is a Tier 2 bare-metal target with prebuilt core libraries available through `rustup`, and `riscv64gc-unknown-linux-gnu` is also a Tier 2 target. The bare-metal target does not provide the full `std` library.
- **Go:** the standard Go compiler uses `GORISCV64=rva20u64` as its baseline for `GOARCH=riscv64`, including `A`, `F`, `D` and `C`. This ISA baseline does not by itself make the stock Go runtime usable in a bare-metal zkVM.
- **C/C++ (GCC, LLVM):** GCC and LLVM support `RV64GC` with the `LP64D` ABI without compiler modifications. Architecture and ABI defaults depend on the toolchain configuration, so builds should select the required ISA and ABI explicitly.

A narrower ISA can require different compiler settings or runtime builds, and some language implementations do not support it. Selecting a common ISA baseline reduces the need for such configurations; it does not eliminate compiler bugs. As noted in the [`Zicclsm` extension](#zicclsm-extension) section, compiler bugs affecting zkVM guests have been observed in practice, and a block that is unprovable because of a miscompilation is a liveness risk for Ethereum.

### Cost to zkVMs

The execution layer's state transition function (STF) does not require floating point arithmetic. `F` and `D` are included for toolchain compatibility rather than an STF arithmetic requirement. Implementations may choose any execution method that preserves the required architectural behavior.

The [Sail RISC-V model](https://github.com/riscv/sail-riscv#supported-risc-v-isa-features) includes `RV64I` and the `M`, `A`, `F`, `D`, `C`, `Zicsr` and `Zifencei` extensions and provides a reference for checking instruction semantics. The platform policies specified here must also be reflected in any verification configuration.

We use 64-bit since many of the algorithms used in the STF can take advantage of a 64-bit word size. For example, U256 integer arithmetic and keccak256.

## Extension requirements

The zkVM executes a single hart in machine mode. The requirements below define the behavior of each extension in this environment. Any behavior not defined here follows the RISC-V Unprivileged ISA specification.

### `A` extension

With a single hart, atomic memory operations (`AMO*`) are equivalent to a load, an operation and a store executed as one instruction.

`LR`/`SC` must be deterministic. The hart initially has no reservation. Each completed `LR` replaces any existing reservation with one covering exactly its addressed word or doubleword. An otherwise valid `SC` succeeds if and only if the most recent `LR` reserved the same address and width and no `SC` has executed since that `LR`. A successful `SC` performs the store and writes 0 to `rd`; a failed `SC` performs no store and writes 1 to `rd`. Every completed `SC` clears the reservation. In this single-hart environment, ordinary loads, stores and `AMO*` instructions do not invalidate the reservation, and spurious `SC` failures are not permitted.

This is a permitted, deterministic restriction of the general `LR`/`SC` rules of the `A` extension. It assumes that no agent other than the hart writes to guest memory during execution.

`Zicclsm` applies to ordinary loads and stores only. A misaligned `LR`, `SC` or `AMO*` must trigger abnormal termination as defined by the [execution termination semantics standard](../standard-termination-semantics/README.md).

### `F` and `D` extensions

Floating point instructions must produce bit-exact results as defined by the RISC-V `F` and `D` extensions, including:

- all five standard rounding modes, selected statically or dynamically through `frm`; executing an instruction with a reserved rounding-mode encoding must raise an illegal-instruction exception;
- the accrued exception flags (`fflags`);
- NaN-boxing of single-precision values in the 64-bit floating point registers.

The canonical-NaN behavior specified by `F` and `D` is required. Instructions that preserve NaN payload bits, including loads, stores and register moves, must preserve them as specified; optional nonstandard NaN-propagation modes are not enabled. Floating point results are therefore deterministic across implementations.

Floating point instructions and the floating point CSRs are available from the first guest instruction, without a guest write to `mstatus.FS`. Implementing privileged floating point context management is not required.

A zkVM may implement these instructions with a dedicated circuit, by replacing them with software routines at transpilation time, or by any other method, as long as the architectural result is the same.

### `C` extension

A zkVM may decode or expand compressed instructions during loading or transpilation, provided it preserves the original guest addresses, instruction lengths, control-flow behavior and link-register values. For example, `C.JALR` writes `pc+2`, not `pc+4`, to the link register.

With the `C` extension, IALIGN is 16: both 16-bit and 32-bit instructions may start at any 2-byte-aligned address. On a taken branch or jump, instruction decoding starts at the effective target address, including when it is the second halfword of a 32-bit instruction decoded from a different address. Normal executable-memory and illegal-instruction checks still apply. `JALR` clears bit 0 of its computed target before transferring control, so the instruction-address-misaligned exception cannot occur on this target.

### `Zicsr` extension

All six `Zicsr` CSR instructions must be implemented for `fflags`, `frm` and `fcsr`, with their specified read/write and side-effect-suppression semantics. Other CSRs, including the `Zicntr` counters (`cycle`, `time`, `instret`), are optional. An access to an unimplemented CSR must raise an illegal-instruction exception.

### `Zifencei` and memory ordering

In this single-hart environment with immutable executable memory and no asynchronously observable memory or I/O operations, `FENCE`, `FENCE.TSO` and `FENCE.I` impose no additional observable behavior. `PAUSE` is treated as a hint with no architectural effect.

### Illegal instructions

An illegal-instruction exception, including an access to an unimplemented CSR or the use of a reserved rounding mode, must trigger abnormal termination as defined by the [execution termination semantics standard](../standard-termination-semantics/README.md).

## `Zicclsm` extension

`Zicclsm` extension is required. It mandates that misaligned loads and stores to main memory regions must be supported. While well-functioning compilers targeting RISC-V typically generate aligned memory accesses, compiler bugs can inadvertently produce unaligned memory operations. Such bugs have been observed in practice across various compiler toolchains. Without `Zicclsm` support, programs that compile and execute correctly on hardware RISC-V implementations would trap or behave incorrectly in zkVM environments. In the context of Ethereum, this creates a critical risk: blocks containing transactions that trigger unaligned accesses due to compiler bugs would become unprovable, potentially halting block production and compromising network liveness.

By requiring `Zicclsm`, zkVMs align with the behavior of physical RISC-V hardware and standard emulators, all of which handle unaligned accesses transparently. This reduces the risk of subtle compatibility issues and ensures that zkVMs remain a reliable execution environment for the broader RISC-V software ecosystem.

`Zicclsm` applies to floating point loads and stores (`FLW`, `FLD`, `FSW`, `FSD` and their compressed forms) in the same way as to integer loads and stores.

zkVMs must provide visibility into the number of unaligned memory accesses that occur during proof generation. This enables developers to monitor unaligned access patterns over time and investigate specific code blocks that trigger them. At minimum, zkVMs should expose a count of unaligned accesses per proof through command-line output or log files, though more granular metrics are encouraged. This observability helps identify potential optimization opportunities and verify that unaligned accesses remain rare as expected, while ensuring the safeguard is working correctly when edge cases do occur.

## zkVM precompiles

Since zkVM precompiles are defined via a C interface, the implementation details of how a zkVM precompile is called does not need to be specified.

## Alternatives considered

### `riscv64im_zicclsm-unknown-none-elf`

An earlier version of this standard proposed `RV64IM` with `Zicclsm`. It minimizes the required ISA, but differs from the baseline of some language implementations, including the standard Go compiler. RV64GC is chosen here to reduce such toolchain differences.

A zkVM may still provide an optimized execution path for programs that use only `RV64IM` instructions. Such instruction streams remain valid on this target when linked with the specified `LP64D` ABI and vendor runtime; arbitrary `LP64` binaries are not automatically compatible.
