# Making-Citeproc-Work-With-Pyinstaller
Citeproc tool in Python to create an executable file using Pyinstaller 
# Making Citeproc Work with PyInstaller

A comprehensive guide and working example for bundling Python applications that use `citeproc-py` with PyInstaller.

## The Problem

When compiling Python scripts that use `citeproc-py` with PyInstaller, you'll encounter errors like:

```
FileNotFoundError: [Errno 2] No such file or directory: 
'C:\\Users\\...\\Temp\\_MEI14402\\citeproc\\data\\locales\\locales.json'
```

or

```
The style apa was not found
```

This happens because PyInstaller only bundles Python code by default, not package data files (like locale files and citation styles).

## Solutions Overview

This repository provides three working solutions:

1. **Bundled Style File Solution** (Recommended) - Most reliable and maintainable
2. **Online Style Download** - Downloads style at runtime
3. **Embedded Style String** - Maximum portability, no external dependencies

---

## Solution 1: Bundled Style File (Recommended)

### Files Needed
- `nbib2jc.py` - Your main script
- `apa.csl` - APA citation style file
- `nbib2jc.spec` - PyInstaller specification file

### Step 1: Download APA Style File

```bash
curl -o apa.csl https://raw.githubusercontent.com/citation-style-language/styles/master/apa.csl
```

Or download manually from: https://github.com/citation-style-language/styles/blob/master/apa.csl

### Step 2: Update Your Python Code

Modify the `format_apa()` function to use the bundled style file:

```python
def format_apa(csl_items):
    """Format citations using APA 7 style."""
    try:
        from citeproc.formatter import plain
        import sys
        
        source = CiteProcJSON(csl_items)
        
        # Get the directory where the script/exe is located
        if getattr(sys, 'frozen', False):
            # Running as compiled executable
            application_path = os.path.dirname(sys.executable)
        else:
            # Running as script
            application_path = os.path.dirname(os.path.abspath(__file__))
        
        apa_style_path = os.path.join(application_path, 'apa.csl')
        
        # Check if file exists
        if not os.path.exists(apa_style_path):
            raise FileNotFoundError(f"APA style file not found at: {apa_style_path}")
        
        style = CitationStylesStyle(apa_style_path, validate=False)
        bibliography = CitationStylesBibliography(style, source, plain)

        for item in csl_items:
            citation = Citation([CitationItem(item["id"])])
            bibliography.register(citation)

        return bibliography
    except Exception as e:
        print(f"❌ Error formatting APA style: {e}")
        raise
```

### Step 3: Create PyInstaller Spec File

Create `nbib2jc.spec`:

```python
# -*- mode: python ; coding: utf-8 -*-

from PyInstaller.utils.hooks import collect_data_files

# Collect citeproc data files (for locales)
datas = collect_data_files('citeproc')

# Add the APA style file
datas += [('apa.csl', '.')]

a = Analysis(
    ['nbib2jc.py'],
    pathex=[],
    binaries=[],
    datas=datas,
    hiddenimports=['citeproc', 'lxml', 'lxml.etree'],
    hookspath=[],
    hooksconfig={},
    runtime_hooks=[],
    excludes=[],
    noarchive=False,
)

pdb = PYZ(a.pure)

exe = EXE(
    pdb,
    a.scripts,
    a.binaries,
    a.datas,
    [],
    name='nbib2jc',
    debug=False,
    bootloader_ignore_signals=False,
    strip=False,
    upx=True,
    upx_exclude=[],
    runtime_tmpdir=None,
    console=True,
)
```

### Step 4: Build

```bash
pyinstaller nbib2jc.spec
```

Your executable will be in the `dist/` folder.

---

## Solution 2: Online Style Download

This solution downloads the style file at runtime. Good for ensuring you always have the latest style.

### Modified format_apa() Function

```python
def format_apa(csl_items):
    """Format citations using APA 7 style."""
    try:
        from citeproc.formatter import plain
        import urllib.request
        import tempfile
        
        source = CiteProcJSON(csl_items)
        
        # Download APA style
        print("📥 Downloading APA style...")
        apa_url = "https://raw.githubusercontent.com/citation-style-language/styles/master/apa.csl"
        
        with tempfile.NamedTemporaryFile(mode='w', suffix='.csl', delete=False, encoding='utf-8') as f:
            response = urllib.request.urlopen(apa_url)
            style_content = response.read().decode('utf-8')
            f.write(style_content)
            apa_style_path = f.name
        
        style = CitationStylesStyle(apa_style_path, validate=False)
        bibliography = CitationStylesBibliography(style, source, plain)

        for item in csl_items:
            citation = Citation([CitationItem(item["id"])])
            bibliography.register(citation)

        # Clean up temp file
        try:
            os.unlink(apa_style_path)
        except:
            pass

        return bibliography
    except Exception as e:
        print(f"❌ Error formatting APA style: {e}")
        raise
```

### PyInstaller Spec File

```python
# -*- mode: python ; coding: utf-8 -*-

from PyInstaller.utils.hooks import collect_data_files

# Only need citeproc data files
datas = collect_data_files('citeproc')

a = Analysis(
    ['nbib2jc.py'],
    pathex=[],
    binaries=[],
    datas=datas,
    hiddenimports=['citeproc', 'lxml', 'lxml.etree'],
    hookspath=[],
    hooksconfig={},
    runtime_hooks=[],
    excludes=[],
    noarchive=False,
)

pdb = PYZ(a.pure)

exe = EXE(
    pdb,
    a.scripts,
    a.binaries,
    a.datas,
    [],
    name='nbib2jc',
    debug=False,
    bootloader_ignore_signals=False,
    strip=False,
    upx=True,
    upx_exclude=[],
    runtime_tmpdir=None,
    console=True,
)
```

**Pros:** Always up-to-date style
**Cons:** Requires internet connection

---

## Solution 3: Embedded Style String

Maximum portability - no external files or internet required.

### Step 1: Download and Read APA Style

Download `apa.csl` and read its contents.

### Step 2: Embed in Code

```python
# Add at the top of your script
APA_STYLE_CSL = """<?xml version="1.0" encoding="utf-8"?>
<style xmlns="http://purl.org/net/xbiblio/csl" class="in-text" version="1.0" demote-non-dropping-particle="never">
  <!-- Paste the full content of apa.csl here -->
  <!-- Due to length, using placeholder - see repository for full version -->
</style>"""

def format_apa(csl_items):
    """Format citations using APA 7 style."""
    try:
        from citeproc.formatter import plain
        import tempfile
        
        source = CiteProcJSON(csl_items)
        
        # Write embedded style to temporary file
        with tempfile.NamedTemporaryFile(mode='w', suffix='.csl', delete=False, encoding='utf-8') as f:
            f.write(APA_STYLE_CSL)
            apa_style_path = f.name
        
        style = CitationStylesStyle(apa_style_path, validate=False)
        bibliography = CitationStylesBibliography(style, source, plain)

        for item in csl_items:
            citation = Citation([CitationItem(item["id"])])
            bibliography.register(citation)

        # Clean up
        try:
            os.unlink(apa_style_path)
        except:
            pass

        return bibliography
    except Exception as e:
        print(f"❌ Error formatting APA style: {e}")
        raise
```

**Pros:** Single file, no dependencies
**Cons:** Larger file size, harder to update style

---

## Quick Start

### Installation

```bash
# Install dependencies
pip install citeproc-py python-docx lxml

# Install PyInstaller
pip install pyinstaller
```

### Build Process

```bash
# Clone repository
git clone https://github.com/yourusername/citeproc-pyinstaller.git
cd citeproc-pyinstaller

# Choose your solution (1 is recommended)
cd solution-1-bundled

# Build
pyinstaller nbib2jc.spec

# Your executable is in dist/nbib2jc.exe (Windows) or dist/nbib2jc (Linux/Mac)
```

---

## Troubleshooting

### Error: "locales.json not found"
**Solution:** Make sure `collect_data_files('citeproc')` is in your spec file's `datas`.

### Error: "The style apa was not found"
**Solutions:**
1. Use Solution 1 and ensure `apa.csl` is in the same directory
2. Check that `datas += [('apa.csl', '.')]` is in your spec file
3. Try Solution 2 (online download) or Solution 3 (embedded)

### Error: "lxml.etree not found"
**Solution:** Add `'lxml.etree'` to `hiddenimports` in your spec file.

### Large executable size
- Use UPX compression (enabled by default in spec file)
- Consider using `--onefile` mode
- Strip debug symbols with `strip=True`

---

## Project Structure

```
citeproc-pyinstaller/
├── README.md
├── solution-1-bundled/
│   ├── nbib2jc.py
│   ├── nbib2jc.spec
│   └── apa.csl
├── solution-2-online/
│   ├── nbib2jc.py
│   └── nbib2jc.spec
├── solution-3-embedded/
│   ├── nbib2jc.py
│   └── nbib2jc.spec
└── examples/
    └── sample.nbib
```

---

## Dependencies

- Python 3.7+
- citeproc-py
- python-docx
- lxml
- PyInstaller

---

## Contributing

Contributions welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Test your changes
4. Submit a pull request

---

## License

MIT License - feel free to use in your projects.

---

## Additional Resources

- [PyInstaller Documentation](https://pyinstaller.org/)
- [citeproc-py Documentation](https://github.com/brechtm/citeproc-py)
- [Citation Style Language](https://citationstyles.org/)
- [CSL Style Repository](https://github.com/citation-style-language/styles)

---

## Credits

Created to solve common issues when packaging Python citation processing tools. Based on real-world NBIB to APA converter requirements.

---

## Support

If you encounter issues:
1. Check the Troubleshooting section
2. Review existing GitHub issues
3. Open a new issue with:
   - Your Python version
   - Your OS
   - Full error traceback
   - PyInstaller version
