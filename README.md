# RISC-V controlled AES-128 / PRESENT-80 shared crypto core

A PicoRV32 RISC-V core controls a resource-shared AES-128 + PRESENT-80 crypto core
(one datapath, `mode` pin: 0 = AES, 1 = PRESENT) through memory-mapped registers.
Verified in QuestaSim against FIPS-197 and the PRESENT-80 reference vectors.

## What the RISC-V does
1. Reads the algorithm choice (mode), key and plaintext from user-input registers.
2. Writes the key and plaintext into the crypto core registers.
3. Writes CTRL (start + mode) and polls STATUS until done.
4. Reads the ciphertext back.
5. (mkfw2) Encrypts a 32-byte message in counter mode in software, decrypts it and self-checks.

## Files and what each one is for
| File | Purpose |
|---|---|
| `soc/soc_top.v` | SoC: PicoRV32 + RAM + shared crypto core + user-input registers + RISC-V activity report |
| `soc/crypto_mmio.v` | Memory-mapped register wrapper for the shared core |
| `soc/tb_all.v` | 2 AES + 2 PRESENT vectors, expected vs actual, PASS/FAIL (`+MODE`, `+TEST`) |
| `soc/tb_self.v` | One vector per mode, lets the whole firmware finish (`-gMODE`) |
| `soc/tb.v` | Simple user-input testbench (`+MODE`) |
| `soc/mkfw.py` | Firmware generator 1: one operation from user input |
| `soc/mkfw2.py` | Firmware generator 2: + cycle count, counter-mode encryption, decrypt, self-check |
| `soc/mkfw3.py` | Firmware generator 3: numbered step log of what the RISC-V does |
| `soc/mkfw4.py` | Firmware generator 4: fixed regression (AES, PRESENT x2, AES) with 600D600D / BAD0C0DE |
| `soc/present_ref.py` | Software PRESENT-80 model to get expected values |
| `hex/*.hex` | Ready-made outputs of mkfw.py ... mkfw4.py |
| `firmware_c/` | C version (start.S, link.ld, main.c, Makefile) for a machine with a RISC-V GCC |

soc_top.v always loads a file named `firmware.hex` from the folder where vsim is started.
To use a ready-made one: `cp ../hex/firmware_simple.hex firmware.hex` (or run `python3 mkfw.py`).
Whichever you put there last is what the RISC-V runs.

## Layout
- `soc/`            SoC top, register wrapper, testbenches, firmware generators
- `final_shared/`   shared architecture RTL (`crypto_top`)        [copy from your design]
- `final_unshared/` unshared architecture RTL (`unshared_crypto`) [copy from your design]
- `aes_rtl/`, `present_RTL/`  sub-modules                         [copy from your design]
- `scripts/collect_rtl.sh`  copies the RTL folders from the server layout

## Memory map
| Address | Block |
|---|---|
| 0x0000_0000 | RAM (firmware, loaded from firmware.hex) |
| 0x4000_1000 | shared crypto core |
| 0x5000_0000 | console + user-input registers (simulation only) |

Crypto registers: CTRL 0x00 (bit0 start, bit1 mode), STATUS 0x04 (bit0 busy, bit1 done),
KEY 0x10-0x1C, PT 0x20-0x2C, CT 0x30-0x3C. First word of each = most significant word.
PRESENT: 80-bit key in KEY[79:0], 64-bit block in PT[63:0] / CT[63:0].

## Setup
PicoRV32 is not included. Download it from https://github.com/YosysHQ/picorv32
and place the folder next to this one as `../picorv32`.
No RISC-V compiler is needed: `soc/mkfw*.py` assemble the firmware directly.

## Run (QuestaSim), from `soc/`
    python3 mkfw.py            # or mkfw2.py (counter mode + self-check) / mkfw3.py (step log)
    vlib work
    vlog ../picorv32/picorv32.v
    vlog ../aes_rtl/aes_sbox.v ../aes_rtl/aes_key_mem.v ../aes_rtl/aes_encipher_block.v ../aes_rtl/aes_core.v ../aes_rtl/aes.v
    vlog ../present_RTL/present_*.v
    vlog ../final_shared/final_shared_substitution.v ../final_shared/final_shared_controller.v ../final_shared/final_shared_datapath.v ../final_shared/final_shared_top.v
    vlog crypto_mmio.v soc_top.v tb_all.v tb_self.v
    vsim -c work.tb_all +MODE=0 +TEST=0      # type: run -all   (then: quit -f)
    vsim -c work.tb_self -gMODE=1            # type: run -all

`tb_all` stops at the first `done` and prints key / plaintext / expected / actual / PASS-FAIL
(use with mkfw.py). `tb_self` lets the whole firmware finish (use with mkfw2.py or mkfw3.py).
`MODE` 0 = AES, 1 = PRESENT; `TEST` 0/1 picks one of two vectors per mode.

## Test vectors
AES-128: FIPS-197 Appendix B and C.1. PRESENT-80: all-ones key / zero plaintext (published)
and a general key/plaintext whose expected value comes from `soc/present_ref.py`.
