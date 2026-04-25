---
name: markdown-to-word
description: >
  Converts Markdown documents to well-formatted Word (.docx) files using python-docx.
  Use this skill when asked to export, convert, or generate a Word document from a Markdown file.
  Produces professional Microsoft-branded styling with Segoe UI fonts, blue-themed headings,
  styled tables with blue headers and alternating row shading, checklist items, and structured metadata.
---

# Markdown to Word Document Conversion

When asked to convert a Markdown file to a Word document, use the Python script at
`~/.copilot/skills/markdown-to-word/md_to_word.py` to perform the conversion.

## Usage

```bash
python ~/.copilot/skills/markdown-to-word/md_to_word.py <input.md> <output.docx>
```

Both arguments are required:
- `<input.md>` — Path to the source Markdown file
- `<output.docx>` — Path for the generated Word document

## Prerequisites

The `python-docx` package must be installed. If it is not available, install it first:

```bash
pip install python-docx
```

## What the script does

1. **Parses** the Markdown file line by line, recognizing:
   - Headings (`#`, `##`, `###`)
   - Tables (pipe-delimited `| col1 | col2 |`)
   - Bullet lists (`- item` or `* item`)
   - Checklist items (`- [ ] item`)
   - Bold field lines (`- **Label:** value`)
   - Italic/instruction text (`_text_`)
   - Blockquotes (`> text`)
   - Horizontal rules (`---`)
   - Code blocks (``` fenced blocks ```)
   - Regular paragraphs

2. **Applies professional formatting:**
   - **Font:** Segoe UI / Segoe UI Semibold
   - **Headings:** Microsoft blue (#0078D4), sized 22pt / 16pt / 13pt for H1 / H2 / H3
   - **Tables:** Blue header row with white text, alternating row shading (#F0F6FC)
   - **Checklist items:** ☐ prefix with left indent
   - **Blockquotes:** Orange-tinted callout text with left indent
   - **Code blocks:** Consolas font with light gray background
   - **Page margins:** 2.54 cm all around

## Customization

If the user requests different styling (colors, fonts, margins), modify the constants at the top of the script:

- `HEADER_BG` — Table header background color (hex without #)
- `ALT_ROW_BG` — Alternating row background color
- `HEADING_COLOR` — RGB tuple for heading text
- `FONT_BODY` / `FONT_HEADING` — Font family names
- `BODY_SIZE` — Body text point size

## Examples

Convert a design review template:
```bash
python ~/.copilot/skills/markdown-to-word/md_to_word.py design-reviews/system-design-template.md "design-reviews/System Design Review Template.docx"
```

Convert any Markdown document:
```bash
python ~/.copilot/skills/markdown-to-word/md_to_word.py README.md output/README.docx
```

## Notes

- The script handles most common Markdown patterns. For highly complex or nested Markdown, minor manual adjustments to the output may be needed.
- Images are not embedded — image references are rendered as placeholder text.
- The script overwrites the output file if it already exists.
