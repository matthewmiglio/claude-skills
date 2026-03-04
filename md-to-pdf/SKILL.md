---
name: md-to-pdf
description: Convert a markdown file to a styled PDF. Handles headings, tables, code blocks, lists, checkboxes, and horizontal rules. The PDF is generated next to the input file. Use when the user wants to convert a .md file to PDF.
allowed-tools: Bash, Read, Write, Edit, Glob, mcp__puppeteer__puppeteer_navigate, mcp__puppeteer__puppeteer_screenshot, mcp__puppeteer__puppeteer_evaluate
user-invocable: true
---

# Markdown to PDF Converter

Convert a `.md` file to a styled, professional PDF using pure Python (`fpdf2`).

## Process

### 1. Parse Arguments

Extract from `$ARGUMENTS`:
- **Input file**: path to the `.md` file to convert

If no file is given, ask the user for the path.

The output PDF is always placed next to the input file with the same name but `.pdf` extension.
Example: `C:/docs/report.md` → `C:/docs/report.pdf`

### 2. Install Dependencies

Run this silently (suppress pip output):
```bash
pip install fpdf2 markdown -q 2>&1 | tail -1
```

### 3. Create Temp Conversion Script

Write the following Python script to a temp file next to the input markdown. Name it `_md_to_pdf_temp.py`.

The script must handle:
- **H1, H2, H3 headers** with bold fonts and underlines
- **Tables** with bordered cells, shaded header rows, and word-wrapped content
- **Fenced code blocks** (```) with monospace font, grey background, and word-wrapped lines
- **Ordered and unordered lists** with indentation
- **Checkbox items** (`- [ ]` and `- [x]`)
- **Horizontal rules** (`---`)
- **Inline markdown stripping** (bold, italic, code, links → plain text)
- **Automatic page breaks** that don't clip content

Font stack (Windows): Arial (`arial.ttf`), Arial Bold (`arialbd.ttf`), Consolas (`consola.ttf`) from `C:/Windows/Fonts/`.

Here is the script to write:

```python
"""Temp script: convert markdown to PDF using fpdf2."""
import re
import sys
from fpdf import FPDF


def flush_table(pdf, rows):
    if not rows:
        return
    parsed = []
    for r in rows:
        cells = [c.strip() for c in r.strip("|").split("|")]
        if all(re.match(r"^[-:]+$", c.strip()) for c in cells):
            continue
        parsed.append(cells)
    if not parsed:
        return
    num_cols = len(parsed[0])
    usable = pdf.w - 20
    col_w = usable / num_cols

    for i, row in enumerate(parsed):
        max_h = 0
        for cell in row:
            pdf.set_font("Body", "", 9)
            lines_needed = max(1, len(pdf.multi_cell(col_w, 5, cell, dry_run=True, output="LINES")))
            h = lines_needed * 5 + 2
            if h > max_h:
                max_h = h
        if pdf.get_y() + max_h > pdf.h - 20:
            pdf.add_page()
        y_before = pdf.get_y()
        for j, cell in enumerate(row):
            x = 10 + j * col_w
            pdf.set_xy(x, y_before)
            if i == 0:
                pdf.set_font("Body", "B", 9)
                pdf.set_fill_color(235, 235, 235)
                pdf.rect(x, y_before, col_w, max_h, "F")
            else:
                pdf.set_font("Body", "", 9)
            pdf.rect(x, y_before, col_w, max_h)
            pdf.set_xy(x + 1, y_before + 1)
            pdf.multi_cell(col_w - 2, 5, cell)
        pdf.set_y(y_before + max_h)
    pdf.ln(4)


def flush_code(pdf, code_lines):
    if not code_lines:
        return
    pdf.set_fill_color(244, 244, 244)
    pdf.set_font("Mono", "", 8)
    code_w = pdf.w - 24
    for line in code_lines:
        line = line.rstrip()
        if pdf.get_y() > pdf.h - 25:
            pdf.add_page()
        pdf.set_x(12)
        pdf.multi_cell(code_w, 4.5, line, fill=True)
    pdf.ln(3)


def strip_inline_md(text):
    text = re.sub(r"\*\*(.+?)\*\*", r"\1", text)
    text = re.sub(r"\*(.+?)\*", r"\1", text)
    text = re.sub(r"`(.+?)`", r"\1", text)
    text = re.sub(r"\[(.+?)\]\(.+?\)", r"\1", text)
    return text


def convert(input_path, output_path):
    with open(input_path, "r", encoding="utf-8") as f:
        lines = f.readlines()

    pdf = FPDF()
    pdf.set_auto_page_break(auto=True, margin=20)
    pdf.add_page()

    pdf.add_font("Body", "", "C:/Windows/Fonts/arial.ttf")
    pdf.add_font("Body", "B", "C:/Windows/Fonts/arialbd.ttf")
    pdf.add_font("Mono", "", "C:/Windows/Fonts/consola.ttf")

    in_code_block = False
    in_table = False
    table_rows = []
    code_lines = []

    for line in lines:
        stripped = line.rstrip()

        # Code block toggle
        if stripped.startswith("```"):
            if in_code_block:
                flush_code(pdf, code_lines)
                code_lines = []
                in_code_block = False
            else:
                if in_table:
                    flush_table(pdf, table_rows)
                    table_rows = []
                    in_table = False
                in_code_block = True
            continue

        if in_code_block:
            code_lines.append(stripped)
            continue

        # Table
        if "|" in stripped and stripped.startswith("|"):
            if not in_table:
                in_table = True
                table_rows = []
            table_rows.append(stripped)
            continue
        else:
            if in_table:
                flush_table(pdf, table_rows)
                table_rows = []
                in_table = False

        # Horizontal rule
        if stripped == "---":
            pdf.ln(2)
            y = pdf.get_y()
            pdf.set_draw_color(200, 200, 200)
            pdf.line(10, y, pdf.w - 10, y)
            pdf.ln(4)
            continue

        # Empty line
        if not stripped:
            pdf.ln(3)
            continue

        clean = strip_inline_md(stripped)

        # Headers
        if stripped.startswith("# "):
            pdf.set_font("Body", "B", 20)
            pdf.multi_cell(0, 9, clean[2:])
            pdf.set_draw_color(50, 50, 50)
            pdf.line(10, pdf.get_y(), pdf.w - 10, pdf.get_y())
            pdf.ln(4)
        elif stripped.startswith("## "):
            pdf.ln(2)
            pdf.set_font("Body", "B", 16)
            pdf.multi_cell(0, 8, clean[3:])
            pdf.set_draw_color(180, 180, 180)
            pdf.line(10, pdf.get_y(), pdf.w - 10, pdf.get_y())
            pdf.ln(3)
        elif stripped.startswith("### "):
            pdf.ln(1)
            pdf.set_font("Body", "B", 13)
            pdf.multi_cell(0, 7, clean[4:])
            pdf.ln(2)
        # Checkbox items (must check before regular list items)
        elif stripped.startswith("- [ ]") or stripped.startswith("- [x]"):
            pdf.set_font("Body", "", 11)
            checked = stripped.startswith("- [x]")
            marker = "[x] " if checked else "[ ] "
            text = strip_inline_md(stripped[6:])
            pdf.set_x(14)
            pdf.multi_cell(pdf.w - 28, 6, marker + text)
        # List items
        elif stripped.startswith("- ") or re.match(r"^\d+\.\s", stripped):
            pdf.set_font("Body", "", 11)
            if stripped.startswith("- "):
                bullet = "- "
            else:
                bullet = re.match(r"^(\d+\.\s)", stripped).group(1)
            text = clean[len(bullet):]
            pdf.set_x(14)
            pdf.multi_cell(pdf.w - 28, 6, bullet + text)
        # Normal text
        else:
            pdf.set_font("Body", "", 11)
            pdf.multi_cell(0, 6, clean)

    # Flush remaining
    if in_table:
        flush_table(pdf, table_rows)
    if in_code_block:
        flush_code(pdf, code_lines)

    pdf.output(output_path)
    print(f"PDF written to {output_path}")


if __name__ == "__main__":
    if len(sys.argv) == 3:
        convert(sys.argv[1], sys.argv[2])
    else:
        print("Usage: python _md_to_pdf_temp.py input.md output.pdf")
```

### 4. Run the Script

```bash
python <dir>/_md_to_pdf_temp.py "<input.md>" "<output.pdf>"
```

Ignore deprecation warnings — only check for actual errors.

### 5. Audit with Puppeteer

Open the generated PDF in puppeteer and screenshot each page to check for:
- Text spilling outside margins
- Overlapping sections
- Awkward page breaks (headers orphaned at bottom of page)
- Tables or code blocks clipped at edges
- Unreadable font sizes

To audit:
1. Navigate to `file:///<output.pdf>`
2. Take a screenshot at 900x1200 to see page 1
3. If multi-page, use `window.scrollTo(0, N)` and take additional screenshots

If issues are found:
1. Edit the temp script to fix them (adjust fonts, margins, wrapping, page break logic)
2. Re-run the script
3. Re-audit until clean

### 6. Cleanup

Once the PDF looks good:
1. Delete the temp script: `rm <dir>/_md_to_pdf_temp.py`
2. Report the output path to the user

## Example

User: `/md-to-pdf C:\docs\report.md`

Result: `C:\docs\report.pdf` created, visually audited, temp script cleaned up.
