# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Educational course platform for "Python Programming - Semester 2" (Ukrainian language). This is a static documentation site built with MkDocs Material, not a Python application. Content focuses on database programming with Python (SQLite, PostgreSQL, Alembic migrations).

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
- **overrides/** — MkDocs Material theme customizations (footer with GitHub issue link).
- **requirements.txt** — MkDocs and related dependencies.

### Content naming convention

Files follow `NN-<topic>-<type>.md` pattern (e.g., `01-sqlite-lecture.md`, `02-sqlite-practice.md`).

### Course modules

1. **Module 1** (implemented): Database work — SQLite, PostgreSQL (psycopg2), Alembic migrations
2. **Module 2** (planned): REST API with Flask
3. **Module 3** (planned): Async programming basics
4. **Module 4** (planned): High-performance async systems

## Key Details

- All documentation is in **Ukrainian**. Maintain this when editing content.
- Code examples in lectures use Python sqlite3, psycopg2, and Alembic libraries.
- Practice assignments build incrementally: weather service → PostgreSQL adaptation → migration integration.
- Site deploys to GitHub Pages at `https://kurotych.com/ua/courses/programming-2sem/`.
- Markdown extensions enabled: `pymdownx.highlight`, `pymdownx.superfences`, `pymdownx.snippets`, `pymdownx.inlinehilite`, `toc` with permalinks.
