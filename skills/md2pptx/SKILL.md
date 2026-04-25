---
name: md2pptx
description: >
  Convert Markdown files to presentation-quality PowerPoint decks with smart
  content planning. Use this skill when the user asks to generate a PowerPoint,
  PPTX, slide deck, or presentation from a Markdown file. Also use when asked
  to "create slides" or "make a presentation" from existing documentation or notes.
---

# Markdown to PowerPoint Converter

This skill converts Markdown (`.md`) files into **presentation-quality** PowerPoint
(`.pptx`) decks using the `md2pptx.py` script bundled in this skill's directory.

Unlike a raw markdown-to-slides dump, this tool **plans** the presentation:
- **Key talking points** go ON the slide (concise, visual)
- **Details, code, overflow content** go into **Speaker Notes**
- Slide count stays tight and focused

## Prerequisites

The `python-pptx` package must be installed:

```bash
pip install python-pptx
```

## How to Use

Run the converter script located in this skill's directory:

```bash
python <skill_dir>/md2pptx.py <input.md> [output.pptx] [--title "Title"]
```

### Arguments

| Argument   | Required | Description                                              |
|------------|----------|----------------------------------------------------------|
| `input`    | Yes      | Path to the input Markdown file                          |
| `output`   | No       | Output `.pptx` path (defaults to same name as input)     |
| `--title`  | No       | Override the presentation title (otherwise uses first H1) |

### Example

```bash
python ~/.copilot/skills/md2pptx/md2pptx.py docs/design.md "Design Review.pptx" --title "Design Review"
```

## How the Planner Works

The converter has a **PresentationPlanner** that analyses each markdown block and
decides what belongs on the slide versus in speaker notes:

| Content Type         | On Slide                          | In Speaker Notes                     |
|----------------------|-----------------------------------|--------------------------------------|
| Bullet lists (≤ 6)   | All items                         | —                                    |
| Bullet lists (> 6)   | First 6 items                     | Remaining items                      |
| Tables (≤ 5 rows)    | Full table                        | —                                    |
| Tables (> 5 rows)    | Header + first 5 rows             | Full table with all rows             |
| Code blocks          | "📝 See speaker notes" callout    | Full code snippet                    |
| Short paragraphs     | Full text                         | —                                    |
| Long paragraphs      | First sentence                    | Full paragraph                       |
| Numbered lists (≤ 6) | All items                         | —                                    |
| Numbered lists (> 6) | First 6 items                     | Remaining items                      |
| Blockquotes          | Callout with accent bar           | —                                    |

## Markdown → Slide Mapping

| Markdown Element | Slide Type / Rendering                             |
|------------------|-----------------------------------------------------|
| `# H1`           | **Title slide** (dark navy background)              |
| `## H2`          | **Section divider** (dark navy background)          |
| `### H3`         | **Content slide title** (white background)          |
| `#### H4+`       | Sub-heading within current content slide            |

## Formatting Support

Inline Markdown formatting is preserved in slides:

- `**bold**` → **bold text**
- `*italic*` → *italic text*
- `` `code` `` → coloured monospace text
- `[link](url)` → underlined blue text

## Theme

The generated presentation uses a professional theme:

- **Widescreen** 16:9 layout (13.333 × 7.5 inches)
- **Dark navy** title and section slides (`#0F1B2D`)
- **White** content slides with navy title bar
- **Microsoft Blue** accent colour (`#0078D4`)
- **Segoe UI** / **Segoe UI Semibold** body/heading fonts
- **Cascadia Code** for code blocks
- Slide numbers on every slide

## Workflow

1. Ensure `python-pptx` is installed (`pip install python-pptx`)
2. Locate the input Markdown file
3. Run the converter: `python ~/.copilot/skills/md2pptx/md2pptx.py input.md output.pptx`
4. Report the number of slides generated and the output file path
5. Remind the user to check **Speaker Notes** in PowerPoint for additional detail
6. If the user wants changes, edit the Markdown and re-run the converter

## Tips

- Structure your Markdown with clear `##` sections for a well-organised deck
- Use `###` headings to create new content slides within a section
- Keep bullet lists concise for the slide; extra items flow to speaker notes
- Code blocks are always sent to speaker notes — keep a summary bullet on-slide
- Tables with many rows are automatically trimmed with overflow in notes
- The converter always appends a "Thank You" closing slide
- **Check Speaker Notes** — that's where the detailed content lives!
