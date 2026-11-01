# Broken Signals

**Category:** Digital forensics / file-format analysis  
**Artifact:** `broken_signal.dat`  
**Status in captured terminal history:** PNG signature repaired; final hidden-data recovery is not shown.

## 1. Initial inspection

```bash
wget https://hicon26-files-b19d71.nbg1.your-objectstorage.com/broken-signals/0a7f6a84e2a6433a/broken-signals.zip
unzip broken-signals.zip
file broken_signal.dat
```

The extracted file was identified only as generic `data`.

## 2. Inspecting the file header

```bash
xxd -l 128 broken_signal.dat
```

The file began with:

```text
00000000: 0000 0000 0d0a 1a0a ...
```

A valid PNG begins with:

```text
89 50 4e 47 0d 0a 1a 0a
```

The first four bytes did not match the PNG signature, while the remaining four bytes matched. The end of the file was also inspected:

```bash
xxd -s -128 broken_signal.dat
```

The dump contained the PNG `IEND` chunk and its expected ending bytes, supporting the hypothesis that this was a PNG with a damaged signature.

## 3. Check for embedded data

```bash
strings -n 4 broken_signal.dat | head -40
binwalk broken_signal.dat
```

`binwalk` identified zlib-compressed data at offset `0x110`, consistent with PNG image data.

## 4. Repair the PNG signature

A copy was made so the original artifact remained unchanged:

```bash
cp broken_signal.dat repaired.png
printf '\x89PNG\r\n\x1a\n' | \
  dd of=repaired.png bs=1 count=8 conv=notrunc
file repaired.png
```

The output identified it as:

```text
PNG image data, 400 x 300, 16-bit/color RGBA, non-interlaced
```

## 5. Result and remaining evidence

The terminal history confirms successful recognition of the repaired PNG. It does **not** include a screenshot of the repaired image, any subsequent steganography extraction, or the final challenge output.

Before publishing this as a fully solved walkthrough, add a screenshot of `repaired.png`, any subsequent extraction/decoding commands, and confirmation of the final recovered artifact.

**Flag:** `hiCTF{REDACTED}`
