# httpcatcher-parser

**httpcatcher-parser** is a Python library and CLI to parse binary **[HTTP Catcher](https://httpcatcher.com)** session
files and export them to the **HAR** format. The project is designed to be installed and
run via **uv** directly from a Git repository or local path (no PyPI required).

> ⚠️ Not affiliated with or endorsed by "HTTP Catcher". Use responsibly.

---

## CLI tools

This README reflects the current `argparse` interfaces in your sources:

### `hc-parser` -- session inspector

Parses a session file and optionally reports unparsed offset ranges ("gaps").

```
Usage: hc-parser SESSION [--outdir DIR] [--report-gaps] [--min-gap-bytes N]

Positional arguments:
  SESSION                 Path to session file to parse

Options:
  --outdir DIR            Output directory. Default: result_{basename_without_suffix}
  --report-gaps           Report unparsed offset ranges and coverage.
  --min-gap-bytes N       Only include gaps >= N bytes in the .gaps.txt report (default 1).
```

### `hc-har` -- HAR exporter

Converts a session file to a `.har` file.

```
Usage: hc-har SESSION [--outdir DIR] [--outfile NAME] [--include-payload] [--no-decompress]

Positional arguments:
  SESSION                 Path to session file

Options:
  --outdir DIR            Output directory (default: result_{basename})
  --outfile NAME          Output HAR file name (default: <basename>.har)
  --include-payload       Include request/response bodies in HAR
  --no-decompress         Do not decompress response bodies
```

---

## Quick start with `uv`

### Run once from Git (no install)
```bash
uvx --from git+https://github.com/mkb79/httpcatcher-parser.git hc-parser --help
uvx --from git+https://github.com/mkb79/httpcatcher-parser.git hc-har --help
```

### Install as a global tool (from Git)
```bash
uv tool install --git https://github.com/mkb79/httpcatcher-parser.git
hc-parser --help
hc-har --help
```

### Local development (working copy)
```bash
# Run without installing
uv run hc-parser path/to/input.session --report-gaps --min-gap-bytes 32
uv run hc-har path/to/input.session --outfile my.har --include-payload

# Install as a tool from the current path
uv tool install --path .
hc-parser --help
```

---

## Exporting sessions from HTTP Catcher

Session files must first be exported from the [HTTP Catcher app](https://httpcatcher.com).

1. Open HTTP Catcher and go to the list of **Saved Sessions**.  
2. Swipe **left** on the desired session.  
3. Tap **Share**.  
4. Choose **Save to Files**.  
5. Select a location in the Files app (e.g. iCloud Drive, On My iPhone).  

You can then pass the saved `session` file to `hc-parser` or `hc-har` as described above.

---
## Project layout

```
.
├─ src/
│  └─ httpcatcher_parser/
│     ├─ hc_parser.py     # binary session parser (argparse)
│     └─ hc_har.py        # HAR exporter (argparse)
├─ Specification.md
├─ README.md
├─ pyproject.toml
└─ .python-version
```

---

## License

AGPL-3.0-only © 2025 [mkb79](mailto:mkb79@hackitall.de)
