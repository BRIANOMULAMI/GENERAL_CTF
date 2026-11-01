# Whispers in the Margin

**Category:** Steganography / document analysis  
**Artifact:** `Those numbers are process counts.docx`  
**Main technique:** Whitespace steganography using spaces and tabs.

## 1. Discover the hidden content

Some paragraphs in the DOCX appeared empty when viewed normally. However, the paragraphs contained combinations of spaces and tab characters. The invisible characters carried encoded information.

## 2. Inspect the DOCX XML

A DOCX file is a ZIP archive containing XML documents. The main document XML was extracted with:

```bash
unzip -p "Those numbers are process counts.docx" word/document.xml
```

## 3. Extract spaces and tabs with Python

```python
import zipfile
import xml.etree.ElementTree as ET

doc = "Those numbers are process counts.docx"

with zipfile.ZipFile(doc) as z:
    root = ET.fromstring(z.read("word/document.xml"))

ns = {
    "w": "http://schemas.openxmlformats.org/wordprocessingml/2006/main"
}

for paragraph in root.findall(".//w:body/w:p", ns):
    text = ""
    for node in paragraph.iter():
        if node.tag.endswith("}t"):
            text += node.text or ""
        elif node.tag.endswith("}tab"):
            text += "\t"

    if text.strip() == "":
        print(repr(text))
```

`repr()` was useful because it made invisible tabs and spaces visible in the terminal.

## 4. Convert whitespace to binary

The observed mapping was:

| Character | Bit |
|---|---|
| Space | `0` |
| Tab | `1` |

The bits were grouped into bytes and converted to ASCII.

The decoded message was:

```text
haha! u got me, now what?
```

This message was used as the Steghide passphrase.

## 5. Extract the hidden file

The recorded extraction command was:

```bash
steghide extract -sf chall.jpg -xf flag.txt \
  -p 'haha! u got me, now what?'
```

The terminal history does not include the resulting file contents or a final extraction confirmation. Add that evidence before presenting this as a fully reproducible end-to-end solution.

**Flag:** `hiCTF{REDACTED}`

## 6. Lessons

- A document that appears blank can still contain meaningful XML content.
- Whitespace characters can encode binary information.
- Inspecting the underlying file format can reveal data hidden by the document viewer.
- `repr()` is a simple way to inspect otherwise invisible whitespace.
