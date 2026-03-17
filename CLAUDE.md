# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Educational course platform for "Python Programming - Semester 2" (Ukrainian language). This is a static documentation site built with MkDocs Material, not a Python application.

## Commands

```bash
# Setup
python3 -m venv env
source env/bin/activate
pip3 install -r requirements.txt

# Serve locally with live reload
mkdocs serve --livereload
```

## Architecture

- **docs/** — Course content in Markdown (Ukrainian). Organized by modules with paired lecture/practice files.
- **mkdocs.yml** — Site configuration, navigation, theme settings, and markdown extensions.
- **overrides/main.html** — Extends base template to add a footer with GitHub issue link.
- **requirements.txt** — MkDocs and related dependencies (pinned versions).

### Content naming convention

Files follow `NN-<topic>-<type>.md` pattern (e.g., `01-sqlite-lecture.md`, `02-sqlite-practice.md`). New content must also be added to the `nav` section in `mkdocs.yml`.

### Course modules

1. **Module 1** (implemented): SQLite, PostgreSQL (psycopg2), Alembic migrations, error handling & transactions, testing, Docker
2. **Module 2** (in progress): HTTP/REST APIs
3. **Module 3** (planned): Async programming basics
4. **Module 4** (planned): High-performance async systems

## Key Details

- All documentation is in **Ukrainian**. Maintain this when editing content.
- Practice assignments build incrementally within each module.
- Site deploys to GitHub Pages at `https://kurotych.com/ua/courses/programming-2sem/`.
- Markdown extensions: `admonition`, `pymdownx.details`, `pymdownx.highlight`, `pymdownx.superfences` (with **mermaid** diagram support), `pymdownx.snippets`, `pymdownx.inlinehilite`, `toc` with permalinks.
- Do not create guides for windows
