# CYBERCONE CTF — Challenge Notes & Write-ups

A learning and portfolio repository documenting my participation in **CYBERCONE CTF** and my investigation of eight challenges.

> **Disclosure note:** Flags are intentionally redacted as `hiCTF{REDACTED}`. Check the event's disclosure rules before publishing challenge solutions, recovered keys, passwords, or detailed walkthroughs.

## Challenges

| Challenge | Area | Main techniques / tools | Documentation status |
|---|---|---|---|
| Broken Signals | Digital forensics / file formats | `file`, `xxd`, `strings`, `binwalk`, PNG signature repair | Partial: transcript confirms header repair; final extraction/result not recorded |
| Ghost in the Wire | Network forensics | PCAP, `capinfos`, TShark, ICMP payload reconstruction | Walkthrough recorded |
| Memory Liar | Memory forensics | `xxd`, memory artifact carving, XOR decoding with Python | Walkthrough reconstructed from transcript |
| Cold Beacon | Reverse engineering | ELF inspection, `objdump`, `.rodata`, Z3, Python | Existing redacted write-up in source notes |
| Epsilon Drift | Reverse engineering | GDB, disassembly, IEEE-754, Python `struct` | Walkthrough recorded |
| Roof of Africa | Steganography | ExifTool, `binwalk`, Steghide | Walkthrough recorded |
| Zanzibar Whisper | Steganography | Steghide metadata and extraction | Walkthrough recorded |
| Whispers in the Margin | Document steganography | DOCX/XML, whitespace-to-binary decoding, Steghide | Partial: password recovery and extraction command recorded; final extraction output not shown |

## Repository layout

```text
.
├── README.md
├── LINKEDIN-POST.md
└── writeups/
    ├── 01-broken-signals.md
    ├── 02-ghost-in-the-wire.md
    ├── 03-memory-liar.md
    ├── 04-cold-beacon.md
    ├── 05-epsilon-drift.md
    ├── 06-roof-of-africa.md
    ├── 07-zanzibar-whisper.md
    └── 08-whispers-in-the-margin.md
```

## Scope and accuracy

These notes were assembled from my terminal history. They preserve commands and observations visible in that record. Where the log does not contain a final result or a complete artifact, the relevant write-up says so instead of inventing missing evidence.

Some challenge-specific artifacts and original solver files were not included in the uploaded transcript. Add them to the repository only after checking that they contain no unredacted flags, credentials, personal paths, or event-sensitive material.

## Tools used

- Linux command line
- Python 3
- `file`, `xxd`, `strings`, `binwalk`, `exiftool`
- Wireshark / TShark / `capinfos`
- `steghide`
- GDB, `objdump`, `readelf`, `checksec`
- Z3

## Flag policy

All flags in this public repository should remain redacted:

```text
hiCTF{REDACTED}
```

Do not commit raw `flag.txt` files, extracted flag artifacts, screenshots containing flags, or unredacted terminal transcripts.
