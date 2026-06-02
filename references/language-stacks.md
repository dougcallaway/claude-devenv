# Language Stack Reference

Add these blocks to `devcontainer.json` `features` and `postCreate.sh` as needed. Mix and match.

---

## Python

**devcontainer.json feature:**
```jsonc
"ghcr.io/devcontainers/features/python:1": {
  "installTools": true,
  "version": "os-provided"   // or "3.12", "3.11", etc.
}
```

**postCreate.sh pattern:**
```bash
PACKAGE_VERSION="1.0"   # pin MAJOR.MINOR; ~= allows patch updates

pip3 install --break-system-packages \
  "some-package~=${PACKAGE_VERSION}"
```

Use `--break-system-packages` on Ubuntu 24.04 (noble) — it's the flag that opts out of PEP 668's protection for system Python. Appropriate inside a container where isolation is already handled by Docker.

Common packages by use case:
- **Data/spreadsheets**: `pandas`, `openpyxl`
- **PDF processing**: `pypdf`, `pdfplumber`, `reportlab`, `pdf2image`, `pytesseract`, `pillow`
- **Document conversion**: `markitdown[pptx]`
- **HTTP clients**: `httpx`, `requests`
- **AI/ML**: `anthropic`, `openai`, `numpy`

---

## Node.js

**devcontainer.json feature:**
```jsonc
"ghcr.io/devcontainers/features/node:1": {
  "version": "lts"   // or "22", "20", etc.
}
```

**postCreate.sh pattern:**
```bash
DOCX_VERSION="^9.5.0"   # ^ allows minor+patch within major

sudo npm install -g \
  "docx@${DOCX_VERSION}"
```

Common global CLIs:
- `@anthropic-ai/claude-code` — Claude Code CLI (already installed by the anthropics feature)
- `docx` — Word document generation
- `pptxgenjs` — PowerPoint generation
- `typescript`, `ts-node` — TypeScript tooling

---

## Go

**devcontainer.json feature:**
```jsonc
"ghcr.io/devcontainers/features/go:1": {
  "version": "latest"   // or "1.22", etc.
}
```

No postCreate step typically needed unless installing specific binaries:
```bash
go install github.com/some/tool@latest
```

---

## Rust

**devcontainer.json feature:**
```jsonc
"ghcr.io/devcontainers/features/rust:1": {
  "version": "latest",
  "profile": "default"
}
```

---

## Multiple Runtimes

Just stack features — devcontainer installs them all:

```jsonc
"features": {
  "ghcr.io/anthropics/devcontainer-features/claude-code:1.0": {},
  "ghcr.io/devcontainers/features/python:1": { "version": "os-provided" },
  "ghcr.io/devcontainers/features/node:1": { "version": "lts" },
  "ghcr.io/devcontainers/features/go:1": { "version": "latest" }
}
```

Feature install order isn't guaranteed, so don't have one feature's postCreate depend on another's output.

---

## System Packages (apt)

Common ones worth knowing:

| Package | Use |
|---------|-----|
| `pandoc` | Document format conversion |
| `poppler-utils` | PDF to image (`pdfimages`, `pdftoppm`) |
| `qpdf` | PDF manipulation |
| `rclone` | Cloud storage sync |
| `tesseract-ocr` | OCR engine (needed by pytesseract) |
| `git`, `curl`, `wget` | Already in most base images |
| `jq` | JSON processing in shell scripts |
| `build-essential` | C compiler for packages with native extensions |
