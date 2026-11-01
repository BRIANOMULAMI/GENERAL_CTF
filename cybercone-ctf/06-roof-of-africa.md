# Roof of Africa

**Category:** Steganography  
**Artifact:** `roof_of_africa.jpg`  
**Main technique:** Extracting embedded data from a JPEG using Steghide.

## 1. Inspect the image

```bash
file roof_of_africa.jpg
binwalk roof_of_africa.jpg
exiftool roof_of_africa.jpg
xxd roof_of_africa.jpg | tail -n 5
```

The file was a 1600 × 1056 JPEG. `binwalk` did not identify an additional embedded file signature. ExifTool confirmed the JPEG metadata, and the file ended with the normal JPEG end marker.

## 2. Check for Steghide content

Steghide was initially unavailable and was installed:

```bash
sudo apt install steghide
```

Then the image was checked:

```bash
steghide info roof_of_africa.jpg
```

The image reported a Steghide capacity of approximately 31.4 KB. Initial passphrase attempts failed.

## 3. Extract the embedded file

The successful command recorded in the terminal history was:

```bash
steghide extract -sf roof_of_africa.jpg -p 5895
```

Steghide reported that it wrote the extracted data to `roof_flag_v2.txt`. The extracted flag is redacted in this public write-up.

## 4. Lessons

- A clean `binwalk` result does not rule out steganography.
- JPEG metadata and file structure are useful first checks, but they do not reveal every embedded payload.
- Steghide can store encrypted data inside supported image formats.
- A correct passphrase is necessary to extract the embedded file.

**Flag:** `hiCTF{REDACTED}`
