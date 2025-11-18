# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Essential Commands

### Running the Application

```bash
# Install dependencies
pip install -r requirements.txt

# Run via main.py (auto-detects GUI vs CLI based on terminal)
python main.py           # GUI if no terminal, CLI if in terminal
python main.py --help    # CLI help in terminal

# Run GUI explicitly
python -m sd_prompt_reader.app

# Run CLI explicitly
python -m sd_prompt_reader.cli -i example.png
python -m sd_prompt_reader.cli -r -i example.png -f JSON -o output.txt
python -m sd_prompt_reader.cli -w -i example.png -m metadata.txt
python -m sd_prompt_reader.cli -c -i example.png -o cleaned.png
```

### Install via pip/pipx
```bash
# Install from PyPI
pip install sd-prompt-reader
# or
pipx install sd-prompt-reader

# Launch GUI
sd-prompt-reader

# Launch CLI
sd-prompt-reader-cli -i example.png
```

### Building Executables

```bash
# macOS: Build .app bundle
python setup.py py2app

# Windows: Build .exe
python setup.py

# Version is read from pyproject.toml and written to __version__.py
# Windows: Creates file_version_info.txt for executable metadata
# macOS: Copies Tcl/Tk libs into .app bundle, updates Poetry dependencies
```

### Code Style

- **Black** code formatting (badge in README)
- Python 3.10+ required (uses `match`/`case` extensively)

## Architecture Overview

### Core Components

The codebase is organized around a detection pipeline that identifies which AI image generation tool created an image and extracts embedded metadata.

**Key directories:**
- `sd_prompt_reader/`: Main package
  - `format/`: Per-format parser modules (A1111, ComfyUI, NovelAI, etc.)
  - `resources/`: Icons and UI assets

**Critical files:**
- `app.py`: Main GUI application (CustomTkinter-based)
- `cli.py`: Command-line interface (Click-based)
- `image_data_reader.py`: Central detection and parsing orchestrator
- `format/base_format.py`: Base class for format parsers
- `format/*.py`: Individual format parsers (a1111, comfyui, novelai, etc.)

### Format Detection Pipeline

The `ImageDataReader` class orchestrates format detection through a heuristic cascade:

1. **TXT files**: If `is_txt=True`, parse as A1111 format directly
2. **SwarmUI legacy EXIF**: Check EXIF tag 0x0110 for JSON with `sui_image_params`
3. **PNG chunk inspection** (PNG only):
   - `parameters` → SwarmUI (if contains `sui_image_params`) or A1111
   - `postprocessing` → A1111 postprocessing
   - `negative_prompt`/`Negative Prompt` → EasyDiffusion
   - `invokeai_metadata` → InvokeAI v3+
   - `sd-metadata` → InvokeAI v2.3-2.3.5
   - `Dream` → InvokeAI v1.x legacy
   - `Software=NovelAI` → NovelAI legacy
   - `prompt` → ComfyUI
   - `Comment` (JSON) → Fooocus
   - `XML:com.adobe.xmp` → DrawThings
   - **RGBA mode + LSB magic** → NovelAI stealth pnginfo
4. **JPEG/WEBP inspection**:
   - `comment` (JSON) → Fooocus
   - **RGBA mode + LSB magic** → NovelAI stealth pnginfo
   - EXIF UserComment → A1111 format in EXIF

**Detection order matters** - parsers are checked sequentially and the first match wins. When adding new formats:
1. Place specific/unique markers early (e.g., EXIF before PNG chunks)
2. Place generic markers late (e.g., `prompt` could match multiple formats)
3. Test with images from multiple formats to prevent false positives

See `image_data_reader.py:54-220` for complete detection flow.

### Parser Architecture

All parsers inherit from `BaseFormat` and implement:
- `.positive`: Positive prompt text
- `.negative`: Negative prompt text
- `.raw`: Raw metadata string
- `.parameter`: Dict of generation parameters (seed, steps, cfg, etc.)
- `.status`: Status enum (READ_SUCCESS, FORMAT_ERROR, etc.)
- `.is_sdxl`: Boolean for SDXL multi-prompt workflows

**Key parsers:**
- `A1111`: Most common format, also used by A1111-compatible tools (default for writing)
- `ComfyUI`: Parses workflow JSON, traverses node graph to find prompts
- `NovelAI`: Handles both legacy and "stealth pnginfo" LSB-encoded metadata
- `InvokeAI`: Three versions (v3+ with `invokeai_metadata`, v2.3-2.3.5 with `sd-metadata`, v1.x with `Dream`)
- `SwarmUI`: Two formats (current JSON in PNG `parameters`, legacy EXIF tag 0x0110)
- `EasyDiffusion`: PNG with `negative_prompt` or `Negative Prompt` fields
- `Fooocus`: JSON in PNG `Comment` field or JPEG `comment` field
- `DrawThings`: XML XMP data in `XML:com.adobe.xmp` with embedded JSON

**Supported image formats:**
- PNG: All formats supported
- JPEG/WEBP: A1111, Fooocus, NovelAI stealth, SwarmUI legacy (EXIF)

## Development Patterns

### Adding a New Format Parser

1. Create `sd_prompt_reader/format/new_format.py`:
```python
from .base_format import BaseFormat

class NewFormat(BaseFormat):
    def __init__(self, info=None, raw=None, width=0, height=0):
        super().__init__(info, raw, width, height)
        # Initialize parser status
        self._status = self.parse()

    def _process(self):
        """Override this method to parse metadata"""
        # Parse from self._info dict (PNG chunks, EXIF, etc.)
        # or from self._raw string
        self._positive = ...  # Required
        self._negative = ...  # Required
        self._setting = ...   # Optional display string

        # Standard parameter keys from BaseFormat.PARAMETER_KEY
        self._parameter = {
            "model": ...,
            "sampler": ...,
            "seed": ...,
            "cfg": ...,
            "steps": ...,
            "size": f"{self._width}x{self._height}"
        }

        # For SDXL multi-prompt workflows
        self._is_sdxl = False  # Set to True if applicable
        self._positive_sdxl = {}  # {"text_g": ..., "text_l": ..., "refiner": ...}
        self._negative_sdxl = {}
```

2. Import in `sd_prompt_reader/format/__init__.py`:
```python
from .new_format import NewFormat
```

3. Add detection logic in `image_data_reader.py:read_data()`:
```python
# Insert at appropriate position in detection cascade
if "new_format_marker" in self._info:
    self._tool = "New Tool Name"
    self._parser = NewFormat(info=self._info, width=self._width, height=self._height)
```

4. Test with CLI:
```bash
python -m sd_prompt_reader.cli -r -i test_image.png -f JSON
python -m sd_prompt_reader.cli -r -i test_image.png  # TXT format
```

**Important:** Place detection logic carefully in the cascade order. Early detection (EXIF, specific markers) prevents false positives. See `image_data_reader.py:54-220` for full detection flow.

### UI State Management

The GUI uses mode enums and `match`/`case` extensively (requires Python 3.10+):

```python
match self.edit_mode:
    case "view":
        # Handle view mode
    case "edit":
        # Handle edit mode
```

Follow this pattern when adding new modes or state transitions. Toggle functions like `edit_mode_switch()` and `edit_mode_update()` handle state changes consistently.

### Logging

Configure logging at module entry points:

```python
from .logger import Logger

logger = Logger("SD_Prompt_Reader.ModuleName")
Logger.configure_global_logger("INFO")  # DEBUG, INFO, WARN, ERROR
```

### Metadata Operations

**Reading:**
```python
from sd_prompt_reader.image_data_reader import ImageDataReader

with open('example.png', 'rb') as f:
    reader = ImageDataReader(f)
    print(reader.positive)  # Positive prompt
    print(reader.negative)  # Negative prompt
    print(reader.parameter) # Dict of generation parameters
    print(reader.status.name)  # READ_SUCCESS, FORMAT_ERROR, etc.
```

**Writing (A1111 format only):**
```python
# Construct metadata from strings
data = ImageDataReader.construct_data(positive, negative, setting)

# Save to image
ImageDataReader.save_image(
    input_path="input.png",
    output_path="output.png",
    format="PNG",
    data=data
)
```

## Technology Stack

- **GUI**: CustomTkinter (modern themed Tkinter), tkinterdnd2 (drag-and-drop)
- **CLI**: Click (command-line interface)
- **Image processing**: Pillow (PIL), piexif (EXIF editing)
- **Platform-specific**: pyobjus/pyobjc (macOS notifications), pefile (Windows PE files)
- **Packaging**: PyInstaller (Windows), py2app (macOS)
- **Dependency management**: Poetry (pyproject.toml) + requirements.txt for pip

### Dependency Notes

- Python 3.10+ required (for `match`/`case` syntax)
- Platform-conditional dependencies in `requirements.txt`:
  - `pyinstaller`, `pyinstaller-hooks-contrib`: Windows only
  - `pyobjus`, `py2app`, `pyobjc-framework-Cocoa`: macOS only
- Entry points defined in `pyproject.toml`:
  - `sd-prompt-reader`: GUI launcher
  - `sd-prompt-reader-cli`: CLI launcher

## Important Constraints

### Format Limitations

**ComfyUI:**
- Stores complete workflow JSON, not just prompts
- Complex/custom node workflows may fail to parse
- Parser traverses node graph for longest branch with valid input/output
- Multiple KSampler nodes result in multiple parameter sets
- SDXL workflows trigger multi-set prompt display mode

**NovelAI:**
- Stealth format uses LSB extraction with magic header `"stealth_pngcomp"`
- Legacy format in PNG chunks
- Only PNG and WEBP supported for stealth format

**TXT files:**
- Only A1111 format supported for import
- Only allowed in edit mode

**Edit mode:**
- Always writes in A1111 format regardless of input format
- All edited images become A1111 format

### Platform-Specific Considerations

**macOS:**
- Unsigned app requires removing quarantine: `xattr -r -d com.apple.quarantine /path/to/app.app`
- Tcl/Tk libs must be copied into .app bundle (handled by `setup.py`)
- Uses py2app with special handling for CustomTkinter resources

**Windows:**
- PyInstaller used for .exe creation
- Version info generated via `file_version_info.txt`
- False positives from antivirus due to PyInstaller (common issue)

**Linux:**
- Requires `python3-tk` package: `sudo apt-get install python3-tk`
- Not regularly tested but should work via pip installation

## Key Files Reference

- `main.py:1-15`: Entry point that auto-detects GUI vs CLI based on terminal (`sys.stdin.isatty()`)
- `app.py:38-600`: Main GUI class with drag-and-drop, viewer components, action buttons
- `cli.py:15-241`: CLI implementation with read/write/clear modes using Click
- `image_data_reader.py:29-285`: Format detection and parsing orchestration
  - `read_data()`: Main detection cascade (lines 54-220)
  - `construct_data()`: Creates A1111 format metadata from strings
  - `save_image()`: Writes metadata to images (PNG chunks or EXIF)
- `format/base_format.py:10-113`: Base parser interface with Status enum
  - `PARAMETER_KEY`: Standard parameter keys ["model", "sampler", "seed", "cfg", "steps", "size"]
  - `Status`: Enum with UNREAD, READ_SUCCESS, FORMAT_ERROR, COMFYUI_ERROR
  - `_process()`: Override this method in subclasses to parse metadata
- `format/comfyui.py`: Most complex parser - workflow graph traversal
- `prompt_viewer.py`: Custom text display widget for prompts with copy/expand/sort features
  - Handles both normal and SDXL multi-set display modes
  - Toggle button for negative prompt visibility
- `parameter_viewer.py`: Grid display for generation parameters
- `constants.py`: UI constants, messages, file paths, supported formats
  - `SUPPORTED_FORMATS = [".png", ".jpg", ".jpeg", ".webp"]`
  - `MESSAGE`: Status messages for various operations
  - `PARAMETER_PLACEHOLDER`: Spacer for empty parameter values

## Recent Changes (feature/explorer branch)

- **File navigation with keyboard shortcuts**: Arrow keys to navigate through images
- **Toggle for negative prompt visibility**: Button to show/hide negative prompts (see `prompt_viewer.py`)
- **Separate CLI executable for Windows**: Standalone CLI .exe in releases
- **NovelAI stealth pnginfo in WEBP format support**: Extended LSB extraction to WEBP

## Common Workflows

### Testing a New Format Parser
```bash
# 1. Get sample image from user
# 2. Check what metadata is embedded
python -m sd_prompt_reader.cli -r -i sample.png -l DEBUG

# 3. Create parser in format/new_format.py
# 4. Add import to format/__init__.py
# 5. Add detection to image_data_reader.py
# 6. Test detection
python -m sd_prompt_reader.cli -r -i sample.png -f JSON

# 7. Test in GUI
python -m sd_prompt_reader.app
```

### Debugging Format Detection
```python
# Add to image_data_reader.py for debugging
self._logger.debug(f"Image format: {f.format}")
self._logger.debug(f"Image mode: {f.mode}")
self._logger.debug(f"PNG info keys: {list(self._info.keys())}")
```

### Working with SDXL Multi-Prompt Workflows
The GUI has two display modes for SDXL workflows with multiple prompt sets:
1. **Separate textboxes**: Clip G, Clip L, Refiner in separate widgets
2. **Tabbed view**: Switch between sets with buttons

Set `self._is_sdxl = True` and populate `self._positive_sdxl`/`self._negative_sdxl` dicts with keys: `"text_g"`, `"text_l"`, `"refiner"`.
