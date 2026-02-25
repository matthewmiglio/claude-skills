---
name: convert-image
description: Convert images between formats (heic, png, jpg, webp, avif, ico, etc.) using the project's image_converter.py script. Supports single file, folder, and in-place conversion modes. Use when the user wants to convert, optimize, or change image formats.
allowed-tools: Bash, AskUserQuestion
---

# Image Converter Skill

Convert images between formats using the Poetry-managed `image_converter.py` script.

**Script location**: `C:\My_Files\my_programs\pixebargain-sites\site-scripts\media\image_converter.py`
**Working directory for commands**: `C:\My_Files\my_programs\pixebargain-sites\site-scripts\media`

## Supported Formats

- **Input**: heic, heif, png, jpg, jpeg, webp, avif, bmp, gif, tiff, tif, ico
- **Output**: png, jpg, jpeg, webp, avif, ico

## Process

### 1. Parse the Request

Extract from `$ARGUMENTS`:
- **Mode**: `single`, `folder`, or `inplace`
- **Input path(s)**: file or directory to convert
- **Output format**: target format (png, jpg, webp, avif)
- **Output directory**: where to save (for single/folder modes)
- **Quality**: custom quality setting (1-100, optional)
- **Source filter**: only convert specific source formats (optional)
- **Overwrite**: whether to overwrite existing files
- **Recursive**: for inplace mode, whether to search subdirectories (default: yes)

### 2. Resolve Missing Information

If the user didn't specify required info, ask using AskUserQuestion:

**Always needed:**
- Input path (file or directory)
- Output format

**Mode logic:**
- If user says "convert this file" → `single` mode, ask for output directory if not given
- If user says "convert all images in folder" with a separate output dir → `folder` mode
- If user says "convert in-place", "replace originals", or "convert and delete originals" → `inplace` mode
- If ambiguous, ask which mode

**Defaults:**
- Quality: use script defaults (jpg=85, webp=85, avif=80) unless user specifies
- Overwrite: off unless user says to overwrite
- Recursive: on for inplace mode unless user says otherwise
- If no output directory specified for single/folder mode, suggest a reasonable one or ask

### 3. Build and Run the Command

All commands MUST be run from the script directory using Poetry:

```bash
cd C:/My_Files/my_programs/pixebargain-sites/site-scripts/media && poetry run python image_converter.py [OPTIONS] <command> [COMMAND-OPTIONS]
```

**Global options** (go before the subcommand):
- `--to <format>` (required): png, jpg, jpeg, webp, avif, ico
- `--overwrite`: overwrite existing output files
- `--quality <N>`: quality for lossy formats (1-100)
- `--from <formats>`: comma-separated source formats to filter (e.g., `png,jpg,heic`)

**Subcommands:**

#### single
```bash
cd C:/My_Files/my_programs/pixebargain-sites/site-scripts/media && poetry run python image_converter.py --to webp single --input "/path/to/image.png" --outdir "/path/to/output"
```
Optional: `--delete-source`

#### folder
```bash
cd C:/My_Files/my_programs/pixebargain-sites/site-scripts/media && poetry run python image_converter.py --to webp folder --input-dir "/path/to/images" --outdir "/path/to/output"
```
Optional: `--delete-source`

#### inplace
```bash
cd C:/My_Files/my_programs/pixebargain-sites/site-scripts/media && poetry run python image_converter.py --to webp inplace --input-dir "/path/to/images"
```
Optional: `--no-recursive`

### 4. Report Results

After the command runs:
1. Show the command output (files converted, sizes, compression ratios)
2. Summarize: how many files converted, total size savings if shown
3. Report any failures

## Examples

User: "convert all pngs in my-site/public/images to webp"
→ inplace mode, --from png, --to webp

User: "convert logo.png to jpg and put it in output/"
→ single mode, --input logo.png, --outdir output/, --to jpg

User: "convert everything in photos/ to webp, keep originals, save to converted/"
→ folder mode, --input-dir photos/, --outdir converted/, --to webp

User: "optimize images in assets/ for web"
→ inplace mode, --to webp (recommend webp for best web performance)

User: "convert heic photos to jpg at quality 95"
→ --from heic, --to jpg, --quality 95

User: "convert logo.png to ico for a favicon"
→ single mode, --input logo.png, --to ico (auto-resizes to max 256x256)

## ICO Format Notes

- ICO files are limited to a maximum of 256x256 pixels. Images larger than this are automatically resized (preserving aspect ratio) during conversion.
- Images are converted to RGBA mode to support transparency.
- ICO does not use quality settings (lossless format).
- Common use case: generating favicons from PNG/WebP source images.
