# Contributing to httpcatcher-parser

Thank you for your interest in contributing! 🎉  
This project is under the **AGPL-3.0-only** license. Please make sure you are comfortable with its terms before contributing.

---

## Development setup

- Python **3.10–3.13** supported
- Package manager: [`uv`](https://docs.astral.sh/uv/)

### Clone and install

```bash
git clone https://github.com/mkb79/httpcatcher-parser.git
cd httpcatcher-parser
uv tool install --path .
```

### Run in dev mode

```bash
uv run hc-parser ./samples/input.session --report-gaps
uv run hc-har ./samples/input.session --outfile out.har --include-payload
```

---

## Coding standards

- **Language:** English only (code, comments, docstrings)
- **Docstrings:** [Google style](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings)
- **Formatting/Linting:** [ruff](https://docs.astral.sh/ruff/) recommended

Example `pyproject.toml` snippet:

```toml
[tool.ruff]
target-version = "py310"
line-length = 100
extend-select = ["I", "UP", "B", "SIM", "PL", "RUF"]
```

---

## Submitting changes

1. Fork the repository and create a feature branch:
   ```bash
   git checkout -b feature/my-change
   ```
2. Make your changes (ensure code is PEP-compliant).
3. Add or update tests if applicable.
4. Commit with a clear message:
   ```bash
   git commit -m "Add HAR option to skip decompression"
   ```
5. Push and open a Pull Request.

---

## Reporting issues

- Use the [GitHub Issues](../../issues) tracker.
- Include:
  - Python version and platform
  - Exact CLI command used
  - Traceback or error log
  - Sample (if possible, anonymized)

---

## Legal

By submitting contributions, you agree that your code will be licensed under the same license as this project: **AGPL-3.0-only**.

---

Thank you for helping improve `httpcatcher-parser`! 🚀
