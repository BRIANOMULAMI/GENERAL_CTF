# Epsilon Drift

**Category:** Reverse engineering  
**Points:** 50  
**Artifact:** `epsilon-drift` — Linux x86-64 ELF  
**Goal:** Construct a 32-byte payload represented by exactly 64 hexadecimal characters.

## 1. Initial inspection

```bash
cd ~/Desktop/CTF/steg/Reverse_Engineering/Epsilon_Drift
file epsilon-drift
checksec --file=epsilon-drift
readelf -h epsilon-drift
readelf -S epsilon-drift
strings -a epsilon-drift
```

The executable was analysed with GDB and disassembly. The program expected a hexadecimal argument and converted it into four 64-bit values.

## 2. Follow the preliminary gate

The caller around `0x401f24` compared the helper result with `1`:

```asm
0x401f1f: call 0x4087f0
0x401f24: cmp  $0x1,%eax
0x401f27: je   0x401f6b
```

The helper at `0x4087f0` copied a global value from `0x522cc8` and subtracted one. At the breakpoint, the observed global value was `2`, so the helper returned `1`.

```gdb
break *0x401f24
run <hex-input>
p/x $eax
x/wx 0x522cc8
x/gx 0x522cc0
```

Observed values:

```text
$eax = 0x1
0x522cc8: 0x00000002
```

This confirmed that execution passed the preliminary gate.

## 3. Inspect parser status and length

```gdb
break *0x401f8f
continue
x/wx $rsp+0x3c
x/wx $rsp+0x38
```

For the first candidate, the parser status was zero, but the length was `0x3e` (62), not `0x40` (64). The first payload was only 31 bytes.

## 4. Understand the four 64-bit fields

The intended payload represented four little-endian 64-bit values:

| Field | Value |
|---|---|
| A | `1.1` |
| B | `2.2` |
| C | `3.3000000000000003` |
| D | IEEE-754 negative zero (`-0.0`) |

The floating-point relationship was `A + B = C`.

IEEE-754 distinguishes positive and negative zero at the bit level:

```text
+0.0 = 0x0000000000000000
-0.0 = 0x8000000000000000
```

The negative-zero value must be packed little-endian.

## 5. Generate the exact payload

```python
import struct

values = [1.1, 2.2, 3.3000000000000003]

data = b''.join(struct.pack('<d', x) for x in values)
data += struct.pack('<Q', 0x8000000000000000)

print(data.hex())
print("length:", len(data.hex()))
print("bytes:", len(data))
```

The corrected output was 64 hexadecimal characters, representing 32 bytes.

## 6. Run the corrected input

The 64-character argument was passed to the executable:

```bash
./epsilon-drift <64-character-hex-payload>
```

The executable accepted the payload and returned the challenge flag. The public copy redacts it.

## 7. Lessons

- Confirm exact input length before debugging deeper validation logic.
- Breakpoints reveal which validation stage is actually reached.
- IEEE-754 comparisons may depend on bit-level details.
- Positive zero and negative zero are distinct encodings.
- `struct.pack('<d', ...)` is useful for little-endian binary64 payload construction.

**Flag:** `hiCTF{REDACTED}`
