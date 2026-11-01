# Cold Beacon

**Category:** Reverse engineering  
**Artifact:** `cold-beacon` — ELF 64-bit x86-64  
**Goal:** Recover the 24-byte key accepted by the executable.

## 1. Program behaviour

```bash
./cold-beacon <key>
```

Incorrect inputs returned `config sealed`. The executable was stripped and non-PIE. Runtime strings suggested a Nim build, including `syncio.nim`, `cmdline.nim`, `paramStr` and `IndexDefect`.

## 2. Initial inspection

```bash
file cold-beacon
checksec --file=cold-beacon
readelf -S cold-beacon
readelf -s cold-beacon
objdump -M intel -d cold-beacon > disasm.txt
objdump -s -j .rodata cold-beacon
```

The binary also contained `/proc/self/status` and `TracerPid` strings. These suggested process-status checks, but they were not required for the key recovery described here.

## 3. Locate the validation routine

The main validation routine was around `0x403cff`, called from around `0x404308`. The caller checked that the supplied key length was `0x18` (24 bytes).

The validation routine referenced static data in `.rodata`:

| Address | Role |
|---|---|
| `0x405fa0` | Expected result bytes |
| `0x405fc0` | Constraint/index records |
| `0x406050` | Number of indices in each record |
| `0x406070` | Operation mode for each record |

There were 23 records. Each selected positions in the 24-byte key and applied a byte-level operation. Addition and multiplication wrapped modulo 256.

## 4. Extract the validation records

```python
from pathlib import Path

b = Path("cold-beacon").read_bytes()

target = b[0x5fa0:0x5fa0 + 23]
table  = b[0x5fc0:0x5fc0 + 23 * 6]
counts = b[0x6050:0x6050 + 23]
modes  = b[0x6070:0x6070 + 23]

for i in range(23):
    record = table[i * 6:(i + 1) * 6]
    mode = modes[i]
    indices = list(record[:counts[i]])

    operation = {
        0: "XOR",
        1: "ADD",
        2: "MUL",
        3: "XOR",
        4: "ADD",
        5: "XOR",
    }.get(mode, f"UNKNOWN({mode})")

    print(f"{i:02}: {operation:3} key[{indices}] = {target[i]:02x}")
```

Examples of extracted constraints:

```text
key[5] XOR key[6] = 0x07
(key[9] + key[10]) mod 256 = 0xa8
(key[10] + key[11]) mod 256 = 0x74
```

Other records used addition, multiplication and XOR across multiple byte positions.

## 5. Model the constraints with Z3

```python
from z3 import *

k = [BitVec(f"k{i}", 8) for i in range(24)]
s = Solver()

for c in k:
    s.add(c >= 0x20, c <= 0x7e)

# Secondary condition transcribed from the routine near 0x403dc4:
s.add(k[12] ^ k[4] == k[20])
```

Using 8-bit bit-vectors was important because it naturally reproduced byte overflow.

The first satisfying candidate did not pass the executable. Candidate keys were tested against the original program, rejected models were excluded, and the search continued.

## 6. Candidate testing

The first recorded candidate was rejected with `config sealed`. A later 24-byte candidate was accepted:

```text
C0ldB34c0n::unl0ck::7a3f
```

The executable then printed its embedded configuration. The flag is redacted.

## 7. Reproduction

```bash
file cold-beacon
checksec --file=cold-beacon
objdump -M intel -d cold-beacon > disasm.txt
objdump -s -j .rodata cold-beacon
python3 solve.py
```

On Kali, Z3 can be installed through the distribution package:

```bash
sudo apt update
sudo apt install python3-z3
```

The original `solve.py` and `pk.py` were present in the challenge directory, but their complete contents were not part of the uploaded terminal-history file. Add the original scripts after reviewing them for secrets and environment-specific paths.

## 8. Lessons

- Stripped binaries can still be analysed with disassembly and static data inspection.
- Static lookup tables can encode most of a validation algorithm.
- Bit-vector widths matter when modelling machine arithmetic.
- A solver only proves the constraints included in the model.
- Always validate candidate inputs against the original executable.

**Flag:** `hiCTF{REDACTED}`
