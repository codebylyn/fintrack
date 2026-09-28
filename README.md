# fintrack

![CI](https://github.com/codebylyn/fintrack/actions/workflows/ci.yml/badge.svg)
![Python](https://img.shields.io/badge/python-3.10%2B-blue)
![License](https://img.shields.io/badge/license-MIT-green)

A small command-line tool that imports bank CSV exports, automatically
categorizes each transaction, and shows you a monthly spending summary,
all stored locally in SQLite. No accounts, no cloud, no tracking.

## Table of contents

- [Features](#features)
- [Demo](#demo)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Usage](#usage)
- [CSV format](#csv-format)
- [How categorization works](#how-categorization-works)
- [How it works](#how-it-works)
- [Project structure](#project-structure)
- [Development](#development)
- [Running in GitHub Codespaces](#running-in-github-codespaces)
- [Troubleshooting](#troubleshooting)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

## Features

- **CSV import** from any bank export with `date`, `description`, and `amount` columns
- **Duplicate detection**, so importing the same file twice never double-counts
- **Automatic categorization** using keyword rules (Transport, Food, Groceries, Bills, Income)
- **Monthly summary** table with totals and transaction counts per category
- **Configurable date format**, so it works with `2026-09-01`, `01/09/2026`, and more
- **Local storage** in a single SQLite file you own
- **Tested and linted**, with `pytest` and `ruff` running in GitHub Actions

## Demo

```text
$ fintrack import data/sample.csv
Imported 7 transactions, skipped 0 duplicates.

$ fintrack summary 2026-09
        Summary for 2026-09
┏━━━━━━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━┓
┃ Category      ┃    Total ┃ Count ┃
┡━━━━━━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━┩
│ Groceries     │   -86.20 │     1 │
│ Bills         │   -64.00 │     1 │
│ Uncategorized │   -20.00 │     1 │
│ Transport     │   -20.90 │     2 │
│ Food          │    -4.75 │     1 │
│ Income        │ 3,000.00 │     1 │
└───────────────┴──────────┴───────┘
```

> Tip: replace this block with a screenshot of your own terminal.
> Save it as `docs/demo.png` and use `![Demo](docs/demo.png)`.

## Requirements

- Python 3.10 or newer
- `pip`

Dependencies (installed automatically): [Typer](https://typer.tiangolo.com/)
for the CLI and
