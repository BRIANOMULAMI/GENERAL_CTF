# Zanzibar Whisper

**Category:** Steganography  
**Artifact:** `zanzibar_whisper.jpg`  
**Main technique:** Identifying and extracting a Steghide payload.

## 1. Inspect the image

```bash
steghide info zanzibar_whisper.jpg
```

Steghide reported:

```text
format: jpeg
capacity: 6.4 KB
embedded file "zanzibar_flag2.txt":
    size: 28.0 Byte
    encrypted: rijndael-128, cbc
    compressed: yes
```

This confirmed that the JPEG contained an encrypted, compressed embedded file.

## 2. Extract the payload

The successful extraction command was:

```bash
steghide extract -sf zanzibar_whisper.jpg
```

After entering the passphrase interactively, Steghide wrote `zanzibar_flag2.txt`. The extracted flag is redacted in this public version.

## 3. Lessons

- `steghide info` can reveal the presence, size and encryption status of an embedded file.
- Extraction can be performed interactively or with a supplied passphrase.
- Keep the original image unchanged and write extracted data to a separate file.

**Flag:** `hiCTF{REDACTED}`
