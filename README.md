[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

# Banshee

> The Banshee (Old Irish spelling ben síde) is a Dark creature native to Ireland. Banshees are malevolent spirits that have the appearance of women and their cries are fatal to anyone that hears them. The Laughing Potion is effective defence against them.

Banshee is a binary-translation-based, instruction-accurate RISC-V simulator for PULP-based manycore systems like the [Snitch-based architectures](https://github.com/pulp-platform/snitch_cluster) and [MemPool](https://github.com/pulp-platform/mempool).

## Requirements

Banshee currently requires Rust version 1.63.0 and LLVM 12.

To install Rust, visit [Rustup](https://rustup.rs/) and choose the correct version of Rust during the installation process.

If you already have Rust installed, get the specific version of the Rust with:

```bash
# Install the correct Rust version
rustup install 1.63.0
```

To get LLVM on Ubuntu, install the following packages:

```bash
sudo apt install llvm-12-dev libclang-common-12-dev zlib1g-dev
```

### Setup at IIS

LLVM is installed globally for internal use at IIS. Make sure to have the following environment variables set to point Banshee to the correct version of LLVM:

<details>
<summary><b>Bash</b></summary>
<p>

```bash
# Set recent compiler for the dependencies
export CC=gcc-9.2.0
export CXX=g++-9.2.0

# Point Banshee to LLVM 12
export LLVM_SYS_120_PREFIX=/usr/pack/llvm-12.0.1-af

# Tell cargo which linker to use
export CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_LINKER=/usr/pack/gcc-9.2.0-af/linux-x64/bin/gcc
```

</p>
</details>

<details>
<summary><b>Fish</b></summary>
<p>

```fish
# Set recent compiler for the dependencies
set -x CC gcc-9.2.0
set -x CXX g++-9.2.0

# Point Banshee to LLVM 12
set -x LLVM_SYS_120_PREFIX /usr/pack/llvm-12.0.1-af

# Tell cargo which linker to use
set -x CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_LINKER /usr/pack/gcc-9.2.0-af/linux-x64/bin/gcc
```

</p>
</details>

## Installation

Banshee can be installed on your system using cargo as follows:

    cargo install --path . -f
    banshee --help

To allow the logging levels `debug` and `trace` make sure to install Banshee in debug mode:

    cargo install --debug --path . -f
    banshee --help

## Usage

    cargo run -- -h

Run a binary as follows:

    cargo run -- path/to/riscv/bin

If you make any changes to `src/runtime.rs` or the `../riscv-opcodes`, run `make` to update the `src/runtime.ll` and `src/riscv.rs` files.

To enable logging output, set the `SNITCH_LOG` environment variable to `error`, `warn`, `info`, `debug`, or `trace`. More detailed [configurations](https://docs.rs/env_logger) are possible.

For larger executable you might encounter segmentation faults due to an insufficient stack size. To increase the stack size of the emulation threads set the `RUST_MIN_STACK` environment variable to the desired number of bytes (default is 2MiB). ([More Information](https://doc.rust-lang.org/std/thread/#stack-size))

### Tracing

Instructions can be traced as they execute using the `--trace` option:

    $ banshee path/to/riscv/bin --trace
    # cycle  hart pc        accesses            dasm
    00000001 0005 80010000  x10=00000005      # DASM(f1402573)
    00000002 0005 80010004  x10:00000005 […]  # DASM(00351293)
    00000003 0005 80010008  x6=20000000       # DASM(20000337)
    00000004 0005 8001000c  x5:00000028 […]   # DASM(006282b3)
    00000005 0005 80010010  x5:20000028 […]   # DASM(00a2a023)
    00000006 0005 80010014                    # DASM(10500073)

Piping the output into `sort` will cause the trace to be sorted by cycle and hartid, which is convenient. Piping into `spike-dasm` will provide further inline disassembly:

    $ banshee path/to/riscv/bin --trace | sort | spike-dasm
    # cycle  hart pc        accesses            dasm
    00000001 0005 80010000  x10=00000005      # csrr    a0, mhartid
    00000002 0005 80010004  x10:00000005 […]  # slli    t0, a0, 3
    00000003 0005 80010008  x6=20000000       # lui     t1, 0x20000
    00000004 0005 8001000c  x5:00000028 […]   # add     t0, t0, t1
    00000005 0005 80010010  x5:20000028 […]   # sw      a0, 0(t0)
    00000006 0005 80010014                    # wfi (args unknown)

**Caution:** Piping the stdout through `spike-dasm` can cause the instruction trace to look delayed with respect to debug and trace logs (which run through stderr), if you have them enabled in `SNITCH_LOG`. This is just a visual artifact.

### Unit Tests

Unit tests are in `tests` and can be compiled and built as follows (compilation requires a riscv toolchain):

    make -C tests
    make test

    # or run an individual test `tests/bin/dummy`
    make test-dummy

An `lldb` session with one of the unit tests can be started as follows:

    # for test `tests/bin/dummy`
    make debug-dummy

### Debugging

You can debug the RISC-V binary execution using GDB. First, execute banshee within GDB:

    gdb --args banshee <banshee args>
    (gdb) b execute_binary
    # respond with "y" to "Make breakpoint pending..."
    (gdb) r

Banshee tries to annotate the translated binary with the PC and disassembly of the original program. This is fairly brittle due to how GDB and debuggers work: The disassembly is annotated as an inlined function. This also means that `n` generally steps over the instruction:

    (gdb) n
    0x00007ffff7fc6003 in execute_binary ()

To ensure you see the instructions annotated, use `ni` to step based on instructions. This is tedious, because you need to step through every x86 instruction that corresponds to a RISC-V instruction in the binary, but ensures you see everything:

    (gdb)
    0x8001001c slli rd=5 rs1=a shamt=3 () at <binary>:8
    8   in <binary>
    (gdb)
    0x80010020 sub rd=2 rs1=2 rs2=5 () at <binary>:9
    9   in <binary>
    (gdb)
    0x00007ffff7fc60bb  9   in <binary>
    (gdb)
    0x80010024 slli rd=5 rs1=5 shamt=6 () at <binary>:10
    10   in <binary>
    (gdb)
    0x80010028 sub rd=2 rs1=2 rs2=5 () at <binary>:11
    11  in <binary>
    (gdb)

The line numbers (8 to 11) indicate that these are the 8th to 11th instructions in the binary, counted in order of appearance in the ELF file.

A more convenient trick to step through the program on RISC-V instruction granularity is to use `n` to step to the next instruction (which places you "in front" of the instruction, not seeing its debug info yet), and then using `s` to step into it.

    (gdb) n
    execute_binary () at binary.riscv:35
    35  in binary.riscv
    (gdb) s
    0x80010088 lui imm20=5f5e rd=d () at binary.riscv:35
    35  in binary.riscv

The arguably best debugging experience is to switch GDB into `layout asm` and use `ni` to step through the instructions. This still requires stepping through every x86 instruction, but the output is comfortably readable.

You can set breakpoints on instructions in the RISC-V binary based on their index in the file (useful if you have a disassembly open and can count the number of instructions):

    (gdb) b binary.riscv:9
    Breakpoint 2 at 0x7ffff7fc60b8: file binary.riscv, line 9.
    (gdb) c
    Breakpoint 2, 0x80010020 () at binary.riscv:9

## Dram Preloading

Banshee supports preloading the DRAM of the target architecture with a binary file. To use this feature, pass the `--file-paths` flag in combination with the `--mem-offsets` flag.
These flags take a comma-separated list of paths and offsets, respectively. The offsets are given as hexadecimal numbers that represent the address location in the DRAM.
In order to, convert your data into a binary file, one can use the `bytearray` and `to_bytes` functions in Python. Make sure that you choose little endian as the byte order.

## Dependencies

Banshee currently requires LLVM 12 to be installed on the system. It is *technically* possible to support multiple LLVM versions through the use of cargo features, but that is not yet implemented.

As a hacky workaround, try changing the `llvm-sys = "120"` line to whatever major version you have times 10, and hope for the best. Be prepared that this breaks the build due to some API changes in LLVM.

## Limitations

- Static translation only at the moment
- Float rounding modes are ignored

## License

Banshee is being made available under the permissive, open-source Apache License 2.0 (`Apache-2.0`). See `LICENSE` for more information.

## Publications

If you use Banshee in your work, you can cite us:

<details>
<summary><a href="https://ieeexplore.ieee.org/abstract/document/9643546"><b>Banshee: A Fast LLVM-Based RISC-V Binary Translator</b></a></summary>
<p>

```
@inproceedings{Riedel2021,
  author={Riedel, Samuel and Schuiki, Fabian and Scheffler, Paul and Zaruba, Florian and Benini, Luca},
  booktitle={2021 IEEE/ACM International Conference On Computer Aided Design (ICCAD)},
  title={Banshee: A Fast {LLVM}-Based {RISC-V} Binary Translator},
  year={2021},
  month=nov,
  pages={1105--1113},
  publisheer={IEEE},
  doi={10.1109/ICCAD51958.2021.9643546}
}
```
