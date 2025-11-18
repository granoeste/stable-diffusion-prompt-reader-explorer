**Purpose**: Provide focused, actionable guidance for AI coding agents working on this repository.

**Big Picture**
- **What**: A small cross-platform GUI + CLI tool to read, display, edit, export, and remove Stable Diffusion prompt metadata embedded in images.
- **Where to look**: GUI in `sd_prompt_reader/app.py`, CLI in `sd_prompt_reader/cli.py`, metadata parsing in `sd_prompt_reader/image_data_reader.py`, and per-format parsers in `sd_prompt_reader/format/`.

**Core components & responsibilities**
- `sd_prompt_reader/app.py`: Main GUI; creates UI controls (PromptViewer, ParameterViewer), drag-and-drop handling, and actions (save, export, clear).
- `sd_prompt_reader/cli.py`: Command-line interface using `click`. Supports `read`, `write`, and `clear` modes and mirrors GUI behaviors for batch processing.
- `sd_prompt_reader/image_data_reader.py`: Central detection pipeline. Opens images, inspects `PIL.Image.info`/EXIF, chooses a parser from `sd_prompt_reader/format/*`, and exposes properties like `raw`, `positive`, `negative`, `setting`, `parameter`, `tool`, and `status`.
- `sd_prompt_reader/format/*.py`: Per-format parser classes implementing the `BaseFormat` API. New formats should implement that interface and be imported in `format/__init__.py`.

**Important patterns and conventions**
- Format discovery is heuristic and order-sensitive. `ImageDataReader.read_data()` checks PNG fields, EXIF, LSB steganography (NovelAI), and file format. Preserve this ordering when adding detection rules.
- Parser interface: parsers return status via `BaseFormat.Status` and expose fields accessed by `ImageDataReader` (e.g., `.positive`, `.negative`, `.raw`, `.parameter`, `.is_sdxl`). Follow the existing parser implementations (see `format/a1111.py`, `format/comfyui.py`).
- Use `ImageDataReader.save_image(...)` to write metadata back to images. To construct metadata from text use `ImageDataReader.construct_data(positive, negative, setting)`.
- Use `Logger` (see `sd_prompt_reader/logger.py`) and call `Logger.configure_global_logger(<LEVEL>)` in modules/entrypoints to set verbosity.
- UI state uses simple mode enums and `match`/`case` extensively (Python 3.10+). Keep new code consistent with `match` patterns used throughout.

**Dev / run / build workflows**
- Quick dev run (install deps first):
  - `pip install -r requirements.txt` (or use `poetry install` — see `pyproject.toml`).
  - GUI: `python -m sd_prompt_reader.app` or use the entrypoint `sd-prompt-reader` (poetry script).
  - CLI: `sd-prompt-reader-cli -i example.png` (or `python -m sd_prompt_reader.cli -i example.png`).
- Packaging:
  - Windows: `python setup.py` runs `PyInstaller` with `win.spec` (see `setup.py` and `build_pyinstaller.bat`).
  - macOS: `python setup.py py2app` path handled in `setup.py` (includes copying Tcl/Tk libs).

**Where to change/add formats**
- Add a new module `sd_prompt_reader/format/<your_format>.py` implementing `BaseFormat`.
- Import the class in `sd_prompt_reader/format/__init__.py` so `ImageDataReader` can pick it up.
- Follow parsing/return patterns in `format/a1111.py` and `format/comfyui.py`. Tests are manual: run `sd-prompt-reader-cli -r -i <file>` to validate detection and output.

**Edge cases & gotchas discovered in code**
- NovelAI stealth format uses LSB extraction and checks a magic header — failures set `FORMAT_ERROR`.
- ComfyUI metadata is not strictly standardized; `ComfyUI` parser traverses stored workflow JSON and may fail for complex or custom nodes. Treat ComfyUI parsing as best-effort.
- Some behavior is platform-specific (e.g., `pyobjus` and py2app for macOS). Conditional dependencies are in `pyproject.toml` and `requirements.txt`.

**Small code examples**
- Read a single image from Python:
  ```py
  from sd_prompt_reader.image_data_reader import ImageDataReader
  with open('example.png','rb') as f:
      d = ImageDataReader(f)
  print(d.tool, d.status.name, d.raw)
  ```
- Write metadata via CLI pattern (internals): use `ImageDataReader.save_image(input_path, output_path, format, data)`

**When you modify UI**
- Follow existing structure in `app.py`: add widgets via the CTk classes and enable/disable via the `edit_mode_switch()` and `edit_mode_update()` patterns to preserve consistent state transitions.

**If you are unsure / next steps**
- If detection fails for a sample file, attach the original image when opening an issue and reference `image_data_reader.py` and the format module you want to extend.
- Ask for guidance before changing the parsing order — small changes can shift detection heuristics for other formats.

Please review and tell me if you want more examples (e.g., step-by-step guide to add a format parser, or an example PR adding a format). 
