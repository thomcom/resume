# Overleaf Integration for Programmatic PDF Generation

## Overview

This document describes methods for programmatically compiling LaTeX documents and retrieving generated PDFs, particularly for integration with CI/CD pipelines and automated workflows.

## Critical Limitation

**Overleaf.com has no public API for downloading compiled PDFs.** The compiled PDF exists only in:
1. The browser-based editor (download button)
2. Read-only share links (opens editor with PDF preview)

There is no direct URL like `overleaf.com/project/[id]/output.pdf` that can be fetched programmatically.

## Practical Workflow for This Project

**Goal:** Iteratively refine `main.tex` with visual feedback from compiled PDF.

### Recommended: Dual-Remote Development

```bash
# Setup (one-time)
cd /path/to/resume
git remote add overleaf https://git.overleaf.com/[PROJECT_ID]
# When prompted: username=git, password=[YOUR_TOKEN]

# Development cycle
vim main.tex                          # Edit locally
git add main.tex && git commit -m "Update resume"
git push overleaf main:master         # Push to Overleaf (triggers compile)
# Open Overleaf in browser to view PDF
git push origin main                  # Push to GitHub (triggers Actions build)
```

### Viewing the Compiled PDF

| Method | Access |
|--------|--------|
| Overleaf Editor | Open project URL, PDF renders in right pane |
| Read-Only Link | Share → Turn on link sharing → Copy "view" link |
| GitHub Actions | Download artifact from Actions tab after build |
| Local Compile | `latexmk -xelatex main.tex` (requires TeX Live) |

### Read-Only Share Link

Create a persistent view link for quick access:
1. Open project in Overleaf
2. Click "Share" button
3. "Turn on link sharing"
4. Copy the "view" URL (read-only)

This link auto-updates when you push changes - view in browser without logging in.

## Available Integration Methods

### 1. Overleaf Git Integration (Recommended for This Project)

Overleaf projects can be treated as Git remotes, enabling clone/push/pull workflows.

**Git URL Format:**
```
https://git.overleaf.com/[PROJECT_ID]
```

**Authentication:**
- Username: `git`
- Password: Authentication token from [Overleaf Account Settings](https://www.overleaf.com/user/settings)
- Tokens expire after 1 year
- Maximum 10 tokens per account
- Generate tokens in Account Settings → Project Synchronization → Git Integration

**Clone Command:**
```bash
git clone https://git.overleaf.com/[PROJECT_ID]
# When prompted: username=git, password=[YOUR_TOKEN]
```

**Credential Storage:**
```bash
git config --global credential.helper store
```

**Limitations:**
- Premium feature (requires paid subscription or institutional access)
- No branch support (single `master` branch only)
- Cannot sync existing Overleaf project with existing GitHub repo

### 2. Local LaTeX Compilation (GitHub Actions)

After cloning from Overleaf or maintaining LaTeX source in GitHub, compile PDFs locally or via CI.

**Recommended GitHub Action: `xu-cheng/latex-action@v4`**

**Basic Workflow:**
```yaml
name: Build LaTeX PDF
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: xu-cheng/latex-action@v4
        with:
          root_file: main.tex
      - uses: actions/upload-artifact@v4
        with:
          name: resume-pdf
          path: main.pdf
```

**Key Parameters:**
| Parameter | Default | Description |
|-----------|---------|-------------|
| `root_file` | (required) | LaTeX file(s) to compile, supports globs |
| `compiler` | `latexmk` | Compiler to use |
| `args` | `-pdf -file-line-error -halt-on-error -interaction=nonstopmode` | Compiler arguments |
| `texlive_version` | `latest` | TeXLive version (2020-2025) |
| `latexmk_use_xelatex` | `false` | Use XeLaTeX engine |
| `latexmk_use_lualatex` | `false` | Use LuaLaTeX engine |
| `latexmk_shell_escape` | `false` | Enable shell escape |
| `extra_system_packages` | | Additional system packages |
| `extra_fonts` | | Custom font paths |

**XeLaTeX Example:**
```yaml
- uses: xu-cheng/latex-action@v4
  with:
    root_file: main.tex
    latexmk_use_xelatex: true
```

### 3. Overleaf CLSI API (Self-Hosted Only)

The Common LaTeX Service Interface provides a REST API for compilation. Only available for self-hosted Overleaf instances.

**Endpoint:**
```
POST /project/<project-id>/compile
```

**Request Format:**
```json
{
  "compile": {
    "options": {
      "compiler": "pdflatex",
      "timeout": 60
    },
    "rootResourcePath": "main.tex",
    "resources": [
      {
        "path": "main.tex",
        "content": "\\documentclass{article}..."
      }
    ]
  }
}
```

**Response:**
```json
{
  "compile": {
    "status": "success",
    "outputFiles": [
      {
        "type": "pdf",
        "url": "http://localhost:3013/project/<id>/output/output.pdf"
      }
    ]
  }
}
```

**Compilers:** `latex`, `pdflatex`, `xelatex`, `lualatex`

**Note:** CLSI runs on port 3013 by default. No public API available on overleaf.com.

### 4. "Open in Overleaf" API (Project Creation Only)

Creates new Overleaf projects from external sources. Does not return compiled PDFs.

**Endpoint:**
```
https://www.overleaf.com/docs
```

**Parameters:**
- `snip_uri`: URL to .tex file or .zip archive
- `snip_uri[]`: Multiple file URLs (array)
- `encoded_snip`: URL-encoded LaTeX source
- `engine`: `latex_dvipdf`, `pdflatex`, `xelatex`, `lualatex`
- `main_document`: Primary .tex file name

**Use Case:** Embedding "Open in Overleaf" buttons in documentation.

## Recommended Workflow for This Project

### Option A: GitHub Actions Compilation (No Overleaf Dependency)

1. Maintain LaTeX source in this Git repository
2. Configure GitHub Actions to compile on push
3. Upload PDF as release artifact or to GitHub Pages

```yaml
name: Build Resume PDF
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: xu-cheng/latex-action@v4
        with:
          root_file: main.tex
      - uses: actions/upload-artifact@v4
        with:
          name: resume
          path: main.pdf
      - name: Create Release
        if: startsWith(github.ref, 'refs/tags/')
        uses: softprops/action-gh-release@v1
        with:
          files: main.pdf
```

### Option B: Overleaf ↔ GitHub Sync

1. Link Overleaf project to GitHub via Overleaf's GitHub Synchronization
2. Push changes from Overleaf to GitHub triggers Actions build
3. PDF generated and stored as artifact

**Setup:**
- In Overleaf: Menu → Synchronization → GitHub
- Link to this repository
- Configure GitHub Actions as above

### Option C: Git Bridge with Local Compilation

1. Clone Overleaf project via Git integration
2. Set up dual remotes (Overleaf + GitHub)
3. Local compilation with `latexmk -pdf main.tex`
4. Push to both remotes

```bash
# Initial setup
git clone https://git.overleaf.com/[PROJECT_ID] resume
cd resume
git remote add github git@github.com:user/resume.git
git remote rename origin overleaf

# Workflow
git pull overleaf master
latexmk -pdf main.tex
git push github main
git push overleaf master
```

## Sources

- [Overleaf Git Integration](https://docs.overleaf.com/integrations-and-add-ons/git-integration-and-github-synchronization/git-integration)
- [Git Authentication Tokens](https://docs.overleaf.com/integrations-and-add-ons/git-integration-and-github-synchronization/git-integration/git-integration-authentication-tokens)
- [Overleaf CLSI Repository](https://github.com/overleaf/clsi)
- [Overleaf Developer API](https://www.overleaf.com/devs)
- [xu-cheng/latex-action](https://github.com/xu-cheng/latex-action)
- [GitHub Actions Marketplace - LaTeX](https://github.com/marketplace/actions/github-action-for-latex)
