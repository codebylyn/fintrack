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
for the CLI and [Rich](https://rich.readthedocs.io/) for terminal tables.

## Installation

```bash
git clone https://github.com/codebylyn/fintrack.git
cd fintrack
pip install -e .
```

Verify the install:

```bash
fintrack --help
```

For development (adds `pytest` and `ruff`):

```bash
pip install -e ".[dev]"
```

## Quick start

```bash
fintrack import data/sample.csv
fintrack summary 2026-09
```

## Usage

### `fintrack import`

Import transactions from a CSV file.

```bash
fintrack import FILE [--date-format FORMAT]
```

| Option | Default | Description |
|--------|---------|-------------|
| `FILE` | required | Path to the CSV file |
| `--date-format` | `%Y-%m-%d` | Python `strptime` format of the date column |

Examples:

```bash
# ISO dates (2026-09-01)
fintrack import statement.csv

# Day/month/year dates (01/09/2026)
fintrack import statement.csv --date-format "%d/%m/%Y"

# Month/day/year dates (09/01/2026)
fintrack import statement.csv --date-format "%m/%d/%Y"
```

Output shows how many rows were added and how many were skipped as duplicates.

### `fintrack summary`

Show spending totals per category for one month.

```bash
fintrack summary YYYY-MM
```

Example:

```bash
fintrack summary 2026-09
```

Categories are sorted with the largest spending first. Expenses are negative
numbers and income is positive.

### Choosing where data is stored

By default, data lives in `~/.fintrack/fintrack.db`. Override it with the
`FINTRACK_DB` environment variable:

```bash
export FINTRACK_DB=/tmp/demo.db
fintrack import data/sample.csv
```

This is useful for testing without touching your real data. The variable lasts
only for the current terminal session.

## CSV format

Your file needs a header row with these exact column names:

| Column | Example | Notes |
|--------|---------|-------|
| `date` | `2026-09-01` | Must match `--date-format` |
| `description` | `Uber trip downtown` | Used for categorization |
| `amount` | `-12.50` | Negative = expense, positive = income. Commas like `1,234.50` are handled |

Example file:

```csv
date,description,amount
2026-09-01,Salary payroll deposit,3000.00
2026-09-02,Uber trip downtown,-12.50
2026-09-03,Corner cafe coffee,-4.75
```

If your bank uses different column names, rename the header row in the CSV
before importing. Configurable column mapping is on the roadmap.

## How categorization works

Each transaction description is lowercased and checked against keyword rules in
`src/fintrack/categorizer.py`. The first matching category wins. If nothing
matches, the transaction is labeled `Uncategorized`.

| Category | Keywords |
|----------|----------|
| Transport | uber, grab, taxi, fuel, gas station |
| Food | restaurant, cafe, coffee, burger, pizza |
| Groceries | supermarket, grocery, market |
| Bills | electric, water, internet, phone |
| Income | salary, payroll, deposit |

To add your own, edit the `DEFAULT_RULES` dictionary:

```python
DEFAULT_RULES = {
    "Transport": ["uber", "grab", "taxi", "fuel", "gas station", "jeepney"],
    # ...
}
```

Rules are applied at import time, so re-import after changing them (see
[Troubleshooting](#troubleshooting) for how to reset the database).

## How it works

1. **Read:** `importer.py` streams rows from the CSV and normalizes dates to
   ISO format (`YYYY-MM-DD`) and amounts to floats.
2. **Categorize:** `categorizer.py` assigns each description a category.
3. **Store:** `db.py` inserts rows into SQLite. A `UNIQUE(date, description, amount)`
   constraint combined with `INSERT OR IGNORE` skips duplicates.
4. **Report:** `db.py` groups transactions by category for a given month using
   `strftime('%Y-%m', date)`, and `cli.py` renders the result with Rich.

Known limitation: two genuinely separate transactions with the same date,
description, and amount (for example, two identical coffees on one day) are
treated as duplicates.

## Project structure

```text
fintrack/
├── .devcontainer/
│   └── devcontainer.json      # Codespaces / dev container setup
├── .github/workflows/
│   └── ci.yml                 # Lint + tests on every push
├── data/
│   └── sample.csv             # Demo data
├── src/fintrack/
│   ├── __init__.py
│   ├── cli.py                 # Typer commands (import, summary)
│   ├── db.py                  # SQLite storage and queries
│   ├── categorizer.py         # Keyword rule engine
│   └── importer.py            # CSV reader
├── tests/
│   └── test_fintrack.py
├── pyproject.toml             # Packaging and tool config
├── LICENSE
└── README.md
```

## Development

Set up:

```bash
pip install -e ".[dev]"
```

Run tests:

```bash
pytest -v
```

Lint:

```bash
ruff check .
```

Auto-fix lint issues:

```bash
ruff check . --fix
```

Tests use pytest's `tmp_path` fixture, so they never touch your real database.

## Running in GitHub Codespaces

1. Click **Code** → **Codespaces** → **Create codespace on main**.
2. Wait for the container to build (dependencies install automatically).
3. In the terminal:
```bash
   fintrack import data/sample.csv
   fintrack summary 2026-09
   pytest
```
4. When finished, stop the codespace to save your free quota:
   click the bottom-left **Codespaces** indicator → **Stop Current Codespace**.

## Troubleshooting

**`fintrack: command not found`**
Run `pip install -e .` again from the repo root.

**`does not appear to be a Python project`**
`pyproject.toml` is missing or you're in the wrong folder. Run `pwd` and
`ls pyproject.toml`.

**`ImportError: cannot import name 'app'`**
`src/fintrack/cli.py` is empty or unsaved. Check with `wc -l src/fintrack/*.py`.

**`No transactions found for YYYY-MM`**
You may have opened a new terminal and lost the `FINTRACK_DB` variable, or
imported into a different month. Re-run the `export` and import again.

**`KeyError: 'date'`**
Your CSV headers must be exactly `date,description,amount`.

**`ValueError: time data ... does not match format`**
Pass the right `--date-format` for your bank's dates.

**Start over with a clean database**

```bash
rm ~/.fintrack/fintrack.db
# or, if you used a custom path:
rm "$FINTRACK_DB"
```

## Roadmap

- [ ] User-editable categorization rules in a YAML file
- [ ] Configurable CSV column mapping for different banks
- [ ] `top-categories` command
- [ ] Export summaries to CSV
- [ ] Monthly budget limits with warnings
- [ ] Charts in the terminal
- [ ] Publish to PyPI

## Contributing

Contributions are welcome.

1. Fork the repo and create a branch: `git checkout -b feature/my-change`
2. Make your changes and add tests
3. Run `pytest` and `ruff check .`
4. Commit with a clear message and open a pull request

## License

Released under the [MIT License](LICENSE).

## Author

Built by [@codebylyn](https://github.com/codebylyn).
