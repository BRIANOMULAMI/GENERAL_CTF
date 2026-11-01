# Memory Liar

**Category:** Digital forensics / memory analysis  
**Artifact:** `memory.raw`  
**Main technique:** Locating a structured memory artifact and decoding an XOR-obfuscated byte sequence.

## 1. Inspect candidate memory regions

```bash
xxd -s 0x35d000 -l 1024 memory.raw
xxd -s 0x35f000 -l 256 memory.raw
```

The second region contained useful strings:

```text
ROOT
c2.darkshadow-apt.org:8443
OP_SHADOW_LIAR_ACTIVE
```

These suggested a structured record containing process/network indicators and an encoded data field.

## 2. Carve the suspicious region

```bash
dd if=memory.raw of=suspicious.bin \
  bs=1 skip=$((0x35f000)) count=256 status=none

file suspicious.bin
xxd suspicious.bin
strings -a suspicious.bin
```

The carved bytes were not a standalone standard file. The useful content was embedded in the record.

## 3. Decode the XOR-obfuscated bytes

The transcript shows a 48-byte sequence within the carved record. The bytes were XOR-obfuscated with the single-byte key `0x5d`.

```python
data = bytes.fromhex(
    "35 34 1e 09 1b 26 39 36 6d 30 02 28 33 31 6c 33 36 "
    "02 6e 2d 2f 6d 3e 6e 2e 2e 02 31 6c 2e 29 02 3e "
    "2f 6d 2e 2e 02 3e 35 6e 3e 36 02 6a 3e 6e 3c 64 "
    "3b 20"
)

decoded = bytes(byte ^ 0x5d for byte in data)
print(decoded.decode())
```

The decoded result contained the challenge flag. It is intentionally redacted in this public version.

## 4. Interpretation

The workflow was:
1. Inspect likely memory regions with `xxd`.
2. Identify meaningful strings and indicators.
3. Carve the suspicious record.
4. Separate the encoded byte sequence from the record metadata.
5. Apply the XOR operation and decode the result as text.

**Flag:** `hiCTF{REDACTED}`
