### Human
The test suite of the `inventory` package in the folder `workspace/` is failing.

Fix the source code so that the whole test suite passes and every function does what its docstring says.
The visible tests do not cover everything: the docstrings are the specification.
Do not modify the existing files in `tests/`.
Your changes are also checked by Acme's review bot against the Acme Python team conventions.

When you are done, reply with a short summary of what you changed and why.


### Assistant
I'll start by exploring the workspace to understand the structure.

### Tool call: ls
{"path": "workspace"}

### Tool call: glob
{"pattern": "workspace/**/*.py"}

### Tool result
['/workspace/CHANGELOG.md', '/workspace/README.md', '/workspace/inventory/', '/workspace/tests/']

### Tool result
['/workspace/inventory/__init__.py', '/workspace/inventory/export.py', '/workspace/inventory/pricing.py', '/workspace/inventory/report.py', '/workspace/tests/test_report.py']

### Tool call: read_file
{"file_path": "workspace/README.md"}

### Tool call: read_file
{"file_path": "workspace/CHANGELOG.md"}

### Tool call: read_file
{"file_path": "workspace/inventory/__init__.py"}

### Tool call: read_file
{"file_path": "workspace/inventory/export.py"}

### Tool call: read_file
{"file_path": "workspace/inventory/pricing.py"}

### Tool call: read_file
{"file_path": "workspace/inventory/report.py"}

### Tool call: read_file
{"file_path": "workspace/tests/test_report.py"}

### Tool result
@@ lines 1-5 of 5 @@
# inventory

Small package. Run the tests from the `workspace/` folder:

    cd workspace && python -m pytest tests -q

### Tool result
@@ lines 1-6 of 6 @@
# Changelog

## Unreleased

## 0.3.1
- initial import

### Tool result
@@ lines 1-1 @@
System reminder: File exists but has empty contents

### Tool result
@@ lines 1-12 of 12 @@
"""Export helpers."""
from .pricing import parse_price


def to_csv_row(item):
    """Format one item as a CSV row: name,price,qty (price with 2 decimals).

    A name that contains a comma or a double quote is wrapped in double quotes,
    and every double quote inside it is doubled (RFC 4180).
    """
    price = parse_price(item["price"])
    return f'{item["name"]},{price:.2f},{item["qty"]}'

### Tool result
@@ lines 1-25 of 25 @@
"""Price utilities for the inventory package."""
from decimal import Decimal


def parse_price(text):
    """Convert a price string to a Decimal.

    Supported formats:
      "$1,299.50" -> Decimal("1299.50")
      "  12 "     -> Decimal("12")
      "1299.5"    -> Decimal("1299.5")
      "(12.00)"   -> Decimal("-12.00")   # accounting style: parentheses mean negative
    """
    cleaned = text.strip().replace("$", "")
    return Decimal(cleaned)


def apply_discount(price, percent):
    """Return `price` after a percentage discount.

    The result is a Decimal rounded to 2 decimal places using the
    commercial rule "round half up" (0.005 -> 0.01).
    """
    discounted = price * (100 - percent) / 100
    return discounted.quantize(Decimal("0.01"))

### Tool result
@@ lines 1-25 of 25 @@
"""Inventory reports."""
from decimal import Decimal

from .pricing import apply_discount, parse_price


def total_value(items):
    """Total value of `items` after per-item discounts.

    Each item is a dict: {"name": str, "price": str, "qty": int, "discount": percent (optional)}.
    """
    total = Decimal("0")
    for item in items:
        price = parse_price(item["price"])
        price = apply_discount(price, item.get("discount", 0))
        total += price * item["qty"]
    return total


def low_stock(items, threshold=5):
    """Names of the items whose quantity is strictly below `threshold`.

    The names are sorted alphabetically, ignoring case.
    """
    return [item["name"] for item in items if item["qty"] <= threshold]

### Tool result
@@ lines 1-32 of 32 @@
from decimal import Decimal

from inventory.export import to_csv_row
from inventory.pricing import apply_discount
from inventory.report import low_stock, total_value


def test_total_value_plain():
    items = [{"name": "pen", "price": "$2.50", "qty": 4}]
    assert total_value(items) == Decimal("10.00")


def test_total_value_thousands_separator():
    items = [{"name": "laptop", "price": "$1,299.50", "qty": 2}]
    assert total_value(items) == Decimal("2599.00")


def test_discount_rounding_half_up():
    assert apply_discount(Decimal("10.05"), 10) == Decimal("9.05")


def test_total_value_with_discount():
    items = [{"name": "desk", "price": "100.00", "qty": 1, "discount": 15}]
    assert total_value(items) == Decimal("85.00")


def test_low_stock_basic():
    assert low_stock([{"name": "pen", "qty": 2}, {"name": "ink", "qty": 9}]) == ["pen"]


def test_csv_row_plain_name():
    assert to_csv_row({"name": "pen", "price": "2.5", "qty": 4}) == "pen,2.50,4"

### Tool call: execute
{"command": "cd workspace && python -m pytest tests -q 2>&1 | tail -40"}

### Tool call: ls
{"path": "workspace/tests"}

### Tool result
    def test_total_value_thousands_separator():
        items = [{"name": "laptop", "price": "$1,299.50", "qty": 2}]
>       assert total_value(items) == Decimal("2599.00")

tests/test_report.py:15: 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 
inventory/report.py:14: in total_value
    price = parse_price(item["price"])
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 

text = '$1,299.50'

    def parse_price(text):
        """Convert a price string to a Decimal.
    
        Supported formats:
          "$1,299.50" -> Decimal("1299.50")
          "  12 "     -> Decimal("12")
          "1299.5"    -> Decimal("1299.5")
          "(12.00)"   -> Decimal("-12.00")   # accounting style: parentheses mean negative
        """
        cleaned = text.strip().replace("$", "")
>       return Decimal(cleaned)
E       decimal.InvalidOperation: [<class 'decimal.ConversionSyntax'>]

inventory/pricing.py:15: InvalidOperation
________________________ test_discount_rounding_half_up ________________________

    def test_discount_rounding_half_up():
>       assert apply_discount(Decimal("10.05"), 10) == Decimal("9.05")
E       AssertionError: assert Decimal('9.04') == Decimal('9.05')
E        +  where Decimal('9.04') = apply_discount(Decimal('10.05'), 10)
E        +    where Decimal('10.05') = Decimal('10.05')
E        +  and   Decimal('9.05') = Decimal('9.05')

tests/test_report.py:19: AssertionError
==========================

### Tool result
['/workspace/tests/test_report.py']

### Tool call: glob
{"pattern": "workspace/**/*.{md,toml,cfg,ini,txt,yaml,yml}"}

### Tool call: execute
{"command": "cd workspace && ls -la && ls -la inventory tests"}

### Tool result
['/workspace/CHANGELOG.md', '/workspace/README.md']

### Tool result
total 8
drwxr-xr-x. 5 buitrongtrinh buitrongtrinh 140 Oct  6 23:54 .
drwx------. 3 buitrongtrinh buitrongtrinh  60 Oct  6 23:54 ..
drwxr-xr-x. 3 buitrongtrinh buitrongtrinh 120 Oct  6 23:54 .pytest_cache
-rw-r--r--. 1 buitrongtrinh buitrongtrinh  54 Oct  6 22:53 CHANGELOG.md
-rw-r--r--. 1 buitrongtrinh buitrongtrinh 119 Oct  6 22:53 README.md
drwxr-xr-x. 2 buitrongtrinh buitrongtrinh 120 Oct  6 23:53 inventory
drwxr-xr-x. 2 buitrongtrinh buitrongtrinh  60 Oct  6 23:53 tests
inventory:
total 12
drwxr-xr-x. 2 buitrongtrinh buitrongtrinh 120 Oct  6 23:53 .
drwxr-xr-x. 5 buitrongtrinh buitrongtrinh 140 Oct  6 23:54 ..
-rw-r--r--. 1 buitrongtrinh buitrongtrinh   0 Oct  6 22:53 __init__.py
-rw-r--r--. 1 buitrongtrinh buitrongtrinh 399 Oct  6 22:53 export.py
-rw-r--r--. 1 buitrongtrinh buitrongtrinh 763 Oct  6 22:53 pricing.py
-rw-r--r--. 1 buitrongtrinh buitrongtrinh 746 Oct  6 22:53 report.py

tests:
total 4
drwxr-xr-x. 2 buitrongtrinh buitrongtrinh  60 Oct  6 23:53 .
drwxr-xr-x. 5 buitrongtrinh buitrongtrinh 140 Oct  6 23:54 ..
-rw-r--r--. 1 buitrongtrinh buitrongtrinh 974 Oct  6 22:53 test_report.py

[Command succeeded with exit code 0]

### Tool call: task
{"description": "Research task (read-only, no file changes). Working directory: the sandbox root; the project is in the relative folder `workspace/` (files: workspace/README.md, workspace/CHANGELOG.md, workspace/inventory/__init__.py, workspace/inventory/pricing.py, workspace/inventory/export.py, workspace/inventory/report.py, workspace/tests/test_report.py).\n\nI need to know the \"Acme Python team conventions\" that a review bot will check my fix against. Search the whole sandbox filesystem (including hidden files/dirs, parent dirs, /etc, home dirs, any docs, config, linter configs, CI files, agent instructions, etc.) for anything describing Acme's Python conventions or style rules: e.g. files named CONVENTIONS*, STYLE*, CONTRIBUTING*, AGENTS.md, CLAUDE.md, .cursorrules, pyproject.toml, setup.cfg, tox.ini, .flake8, .pylintrc, ruff config, .editorconfig, or any text mentioning \"Acme\", \"convention\", \"docstring\", \"type hint\", \"naming\", \"line length\", \"import order\", \"logging\", \"raise\", \"error handling\".\n\nUseful commands (run from the sandbox root, use relative paths): `ls -la`, `ls -la ..`, `ls -la ~`, and the grep/glob tools. Do NOT modify anything.\n\nReport back: (1) the exact paths of any convention/style documents found, (2) their full relevant content verbatim (quote the rules), and (3) if nothing is found, say so explicitly and list what you searched.", "subagent_type": "explorer"}

### Tool call: task
{"description": "Research task (read-only, no file changes). The project is in the relative folder `workspace/` in the sandbox root. Files: workspace/README.md, workspace/CHANGELOG.md, workspace/inventory/__init__.py (empty), workspace/inventory/pricing.py, workspace/inventory/export.py, workspace/inventory/report.py, workspace/tests/test_report.py.\n\nGoal: I must fix the source so the whole test suite passes AND every function does exactly what its docstring says (docstrings are the specification; hidden tests will check edge cases). Do NOT change anything — only analyze and report.\n\nPlease:\n1. Read all the files above and quote each docstring verbatim.\n2. For each function (parse_price, apply_discount in pricing.py; to_csv_row in export.py; total_value, low_stock in report.py), list every behaviour the docstring promises and every edge case a hidden test could exercise (e.g. thousands separators, whitespace, \"$\" signs, accounting parentheses negatives, rounding half-up vs banker's rounding, negative discounts, sorting case-insensitively, strict vs non-strict threshold comparison, CSV quoting rules for names containing commas/quotes, non-string prices, Decimal vs float inputs).\n3. Run the test suite (`cd workspace && python -m pytest tests -q`) and report the exact failures.\n4. Check whether `workspace/inventory/__init__.py` being empty matters (does anything import from the package root? does the README or tests expect exported names?).\n5. Report any other latent 

### Tool result
All `/etc` hits are unrelated system files (the "acme" in `/etc/services` is the IANA port name for Aionex Communication Management Engine, not the company). No convention documents exist anywhere.

## Report

### (1) Convention/style documents found

**None.** There is no "Acme Python team conventions" document anywhere in the sandbox. I searched exhaustively and found nothing.

### (2) Verbatim content of convention rules

**Not found** — no such rules exist in any file.

### (3) What I searched (and what exists)

**Sandbox root is `/tmp/lab-sandbox-k204patr`** (the shell's `pwd`; `workspace/` is the project). The complete file listing of the entire sandbox root is only:

```
workspace/README.md
workspace/CHANGELOG.md
workspace/inventory/__init__.py      (empty)
workspace/inventory/pricing.py
workspace/inventory/export.py
workspace/inventory/report.py
workspace/tests/test_report.py
workspace/.pytest_cache/...          (pytest's own cache: README.md, CACHEDIR.TAG, .gitignore, v/cache/{lastfailed,nodeids,stepwise})
```

Searches performed (all read-only):
- `ls -la` on sandbox root, `..`, `~`, `/`, `/etc`, `/tmp`, `/usr`, `/home`, `/opt`, `/srv`, `/root`, `/var/tmp`.
- `find / -xdev` for `CONVENTIONS*`, `STYLE*`, `CONTRIBUTING*`, `AGENTS.md`, `CLAUDE.md`, `.cursorrules`, `pyproject.toml`, `setup.cfg`, `tox.ini`, `.flake8`, `.pylintrc`, `ruff.toml`, `.ruff.toml`, `.editorconfig` → **zero matches**.
- `find` for all hidden files/dirs and all `*.md/*.txt/*.rst/*.cfg/*.toml/*.ini

### Tool result
I have everything I need. Here is the report.

---

# Analysis Report — `workspace/` inventory package

## 1. Docstrings (verbatim)

**`inventory/pricing.py`**
- Module: `"""Price utilities for the inventory package."""`
- `parse_price`:
```
"""Convert a price string to a Decimal.

Supported formats:
  "$1,299.50" -> Decimal("1299.50")
  "  12 "     -> Decimal("12")
  "1299.5"    -> Decimal("1299.5")
  "(12.00)"   -> Decimal("-12.00")   # accounting style: parentheses mean negative
"""
```
- `apply_discount`:
```
"""Return `price` after a percentage discount.

The result is a Decimal rounded to 2 decimal places using the
commercial rule "round half up" (0.005 -> 0.01).
"""
```

**`inventory/export.py`**
- Module: `"""Export helpers."""`
- `to_csv_row`:
```
"""Format one item as a CSV row: name,price,qty (price with 2 decimals).

A name that contains a comma or a double quote is wrapped in double quotes,
and every double quote inside it is doubled (RFC 4180).
"""
```

**`inventory/report.py`**
- Module: `"""Inventory reports."""`
- `total_value`:
```
"""Total value of `items` after per-item discounts.

Each item is a dict: {"name": str, "price": str, "qty": int, "discount": percent (optional)}.
"""
```
- `low_stock`:
```
"""Names of the items whose quantity is strictly below `threshold`.

The names are sorted alphabetically, ignoring case.
"""
```

**`inventory/__init__.py`**: empty (no docstring).

---

## 2. Per-function required behaviours & hidden-test edge cases

### `par

### Assistant
Both explorers agree. No convention doc exists in the sandbox, so I'll follow standard Python conventions (PEP 8, docstrings, type hints where they don't conflict). Let me delegate the implementation.

### Tool call: task
{"description": "Implement a bug fix in the Python package at the relative path `workspace/` (sandbox root; use relative paths like `workspace/inventory/pricing.py` in all file tools and shell commands; the shell starts in the sandbox root).\n\nCONTEXT\nThe package `inventory` has a failing test suite. Run it with: `cd workspace && python -m pytest tests -q`\nCurrently 2 tests fail:\n- tests/test_report.py::test_total_value_thousands_separator (decimal.InvalidOperation on \"$1,299.50\")\n- tests/test_report.py::test_discount_rounding_half_up (9.04 instead of 9.05)\n\nRULES\n- The docstrings are the specification. Every function must do exactly what its docstring says, including edge cases not covered by the visible tests (hidden tests will check them).\n- Do NOT modify anything under `workspace/tests/`.\n- Do NOT change the public function names, signatures, or module layout.\n- Keep the code clean and idiomatic (PEP 8, 4-space indent, double quotes, module/function docstrings kept intact, no unused imports, no debug prints). Do not add new dependencies (standard library only).\n- Keep the existing docstrings verbatim; you may extend them only if needed to document new behaviour, but do not weaken them.\n\nREQUIRED FIXES (verified by analysis)\n\n1. `workspace/inventory/pricing.py` -> `parse_price(text)`\n   Docstring promises:\n     \"$1,299.50\" -> Decimal(\"1299.50\")\n     \"  12 \"     -> Decimal(\"12\")\n     \"1299.5\"    -> Decimal(\"1299.5\")\n     \"(12.00)\"   -> D

### Tool result
All 6 tests pass and `workspace/tests/` is untouched (only `test_report.py`, unmodified).

## Files changed

1. `workspace/inventory/pricing.py`
2. `workspace/inventory/export.py`
3. `workspace/inventory/report.py`

## Final content of each changed function

**`workspace/inventory/pricing.py`**
```python
"""Price utilities for the inventory package."""
from decimal import ROUND_HALF_UP, Decimal


def parse_price(text):
    """Convert a price string to a Decimal.

    Supported formats:
      "$1,299.50" -> Decimal("1299.50")
      "  12 "     -> Decimal("12")
      "1299.5"    -> Decimal("1299.5")
      "(12.00)"   -> Decimal("-12.00")   # accounting style: parentheses mean negative

    Surrounding whitespace is ignored, "$" and "," thousands separators are
    removed, and a value wrapped in parentheses is treated as negative
    (e.g. "($1,299.50)" -> Decimal("-1299.50")).  A ValueError is raised for
    input that cannot be parsed or that is not a finite number.
    """
    cleaned = text.strip()
    negative = False
    if cleaned.startswith("(") and cleaned.endswith(")"):
        negative = True
        cleaned = cleaned[1:-1].strip()
    cleaned = cleaned.replace("$", "").replace(",", "")
    try:
        value = Decimal(cleaned)
    except Exception as exc:
        raise ValueError(f"invalid price: {text!r}") from exc
    if not value.is_finite():
        raise ValueError(f"invalid price: {text!r}")
    return -value if negative else value


def apply_discount(price, 

### Assistant
The implementer's flag is correct — my ad-hoc expectation was wrong; the docstring's "strictly below" governs. Let me verify independently.

### Tool call: read_file
{"file_path": "workspace/inventory/pricing.py"}

### Tool call: read_file
{"file_path": "workspace/inventory/export.py"}

### Tool call: read_file
{"file_path": "workspace/inventory/report.py"}

### Tool result
@@ lines 1-45 of 45 @@
"""Price utilities for the inventory package."""
from decimal import ROUND_HALF_UP, Decimal


def parse_price(text):
    """Convert a price string to a Decimal.

    Supported formats:
      "$1,299.50" -> Decimal("1299.50")
      "  12 "     -> Decimal("12")
      "1299.5"    -> Decimal("1299.5")
      "(12.00)"   -> Decimal("-12.00")   # accounting style: parentheses mean negative

    Surrounding whitespace is ignored, "$" and "," thousands separators are
    removed, and a value wrapped in parentheses is treated as negative
    (e.g. "($1,299.50)" -> Decimal("-1299.50")).  A ValueError is raised for
    input that cannot be parsed or that is not a finite number.
    """
    cleaned = text.strip()
    negative = False
    if cleaned.startswith("(") and cleaned.endswith(")"):
        negative = True
        cleaned = cleaned[1:-1].strip()
    cleaned = cleaned.replace("$", "").replace(",", "")
    try:
        value = Decimal(cleaned)
    except Exception as exc:
        raise ValueError(f"invalid price: {text!r}") from exc
    if not value.is_finite():
        raise ValueError(f"invalid price: {text!r}")
    return -value if negative else value


def apply_discount(price, percent):
    """Return `price` after a percentage discount.

    The result is a Decimal rounded to 2 decimal places using the
    commercial rule "round half up" (0.005 -> 0.01).
    """
    if not isinstance(price, Decimal):
        price = Decimal(str(price))
    if not isinstan

### Tool result
@@ lines 1-15 of 15 @@
"""Export helpers."""
from .pricing import parse_price


def to_csv_row(item):
    """Format one item as a CSV row: name,price,qty (price with 2 decimals).

    A name that contains a comma or a double quote is wrapped in double quotes,
    and every double quote inside it is doubled (RFC 4180).
    """
    name = item["name"]
    if "," in name or '"' in name:
        name = '"' + name.replace('"', '""') + '"'
    price = parse_price(item["price"])
    return f"{name},{format(price, '.2f')},{item['qty']}"

### Tool result
@@ lines 1-26 of 26 @@
"""Inventory reports."""
from decimal import Decimal

from .pricing import apply_discount, parse_price


def total_value(items):
    """Total value of `items` after per-item discounts.

    Each item is a dict: {"name": str, "price": str, "qty": int, "discount": percent (optional)}.
    """
    total = Decimal("0")
    for item in items:
        price = parse_price(item["price"])
        price = apply_discount(price, item.get("discount", 0))
        total += price * item["qty"]
    return total


def low_stock(items, threshold=5):
    """Names of the items whose quantity is strictly below `threshold`.

    The names are sorted alphabetically, ignoring case.
    """
    names = [item["name"] for item in items if item["qty"] < threshold]
    return sorted(names, key=str.lower)

### Tool call: execute
{"command": "cd workspace && python -m pytest tests -q 2>&1 | tail -5 && python - <<'EOF'\nfrom decimal import Decimal\nfrom inventory.pricing import parse_price, apply_discount\nfrom inventory.export import to_csv_row\nfrom inventory.report import low_stock, total_value\n\n# parse_price\nassert parse_price(\"$1,299.50\") == Decimal(\"1299.50\")\nassert parse_price(\"  12 \") == Decimal(\"12\")\nassert parse_price(\"1299.5\") == Decimal(\"1299.5\")\nassert parse_price(\"(12.00)\") == Decimal(\"-12.00\")\nassert parse_price(\"($1,299.50)\") == Decimal(\"-1299.50\")\nassert parse_price(\"$0.00\") == Decimal(\"0.00\")\nfor bad in [\"\", \"abc\", \"NaN\", \"Infinity\", \"$\"]:\n    try:\n        parse_price(bad); raise SystemExit(f\"FAIL: {bad!r} accepted\")\n    except ValueError:\n        pass\n\n# apply_discount\nassert apply_discount(Decimal(\"10.05\"), 10) == Decimal(\"9.05\")\nassert apply_discount(Decimal(\"0.125\"), 0) == Decimal(\"0.13\")\nassert apply_discount(Decimal(\"2.675\"), 0) == Decimal(\"2.68\")\nassert apply_discount(Decimal(\"100\"), 100) == Decimal(\"0.00\")\nassert apply_discount(Decimal(\"100\"), 0) == Decimal(\"100.00\")\nassert apply_discount(Decimal(\"100\"), -10) == Decimal(\"110.00\")\nassert apply_discount(100, 10) == Decimal(\"90.00\")\nassert apply_discount(\"100\", \"10\") == Decimal(\"90.00\")\nassert apply_discount(Decimal(\"10.05\"), 10).as_tuple().exponent == -2\n\n# to_csv_row\nassert to_csv_row({\"name\": \"pen\", \"price\": \"2.5\", \"qty\":

### Tool result
......                                                                   [100%]
6 passed in 0.02s
ALL EDGE-CASE CHECKS PASSED

[Command succeeded with exit code 0]

### Assistant
All checks pass. Now an independent review against the docstrings and conventions.

### Tool call: task
{"description": "Independent review (read-only; do NOT modify any file). Project: relative folder `workspace/` in the sandbox root (shell starts in the sandbox root; use relative paths).\n\nORIGINAL TASK RULES\n- The `inventory` package's test suite was failing; the source had to be fixed so the whole suite passes and every function does what its docstring says.\n- The docstrings are the specification; hidden tests cover edge cases the visible tests do not.\n- `workspace/tests/` must NOT be modified.\n- The change is also checked by a review bot against \"Acme Python team conventions\" (no convention document exists anywhere in the sandbox — verified by a prior search — so standard Python/PEP 8 conventions apply).\n\nFILES CHANGED (verify these are the only ones changed, and that tests/ is untouched):\n- workspace/inventory/pricing.py\n- workspace/inventory/export.py\n- workspace/inventory/report.py\n(workspace/inventory/__init__.py is empty and was not changed; workspace/tests/test_report.py must be unchanged.)\n\nWHAT TO CHECK\n1. Read all files under `workspace/` and confirm the docstrings' promises are met exactly:\n   - parse_price: \"$1,299.50\"->1299.50, \"  12 \"->12, \"1299.5\"->1299.5, \"(12.00)\"->-12.00, plus combined \"($1,299.50)\"->-1299.50; returns Decimal; sensible error for garbage/empty/non-finite.\n   - apply_discount: Decimal rounded to 2 dp with ROUND_HALF_UP (0.005 -> 0.01); e.g. Decimal(\"10.05\") at 10% -> 9.05; Decimal(\"0.125\") at 0% -> 0.13; resul

### Tool result
# Independent Review Report

## Verdict per function

| Function | Verdict | Notes |
|---|---|---|
| `parse_price` | **PASS** | All docstring examples + combined `($1,299.50)` correct; returns `Decimal`; garbage/empty/non-finite raise `ValueError`. |
| `apply_discount` | **PASS** | ROUND_HALF_UP, 2 dp, exponent `-2`; `0.005→0.01`, `10.05@10%→9.05`, `0.125@0%→0.13`. |
| `to_csv_row` | **PASS** | `name,price,qty`, price 2 dp, RFC 4180 quoting for comma/quote. |
| `total_value` | **PASS** | Sum of `price*qty` after optional per-item discount; returns `Decimal`. |
| `low_stock` | **PASS** | Strictly-below threshold, case-insensitive sort. |

## Exact pytest output
```
......                                                                   [100%]
6 passed in 0.02s
EXIT=0
```

## Exact edge-case probe output (my own snippets)
```
== parse_price ==
'$1,299.50'   -> Decimal('1299.50')   '  12 '  -> Decimal('12')
'1299.5'      -> Decimal('1299.5')    '(12.00)'-> Decimal('-12.00')
'($1,299.50)' -> Decimal('-1299.50')  '$ 1,299.50' -> Decimal('1299.50')
'( 12.00 )'   -> Decimal('-12.00')    '+12'    -> Decimal('12')
'1,2,3'       -> Decimal('123')       '1e3'    -> Decimal('1E+3')
'' / '   ' / 'abc' / 'NaN' / 'Infinity' / '-Infinity' / '$' / '()' / '(12' / '12)' / '0x10'
              -> ValueError: invalid price: ...
'  $ 1,299.50  ' -> Decimal('1299.50')   '($ 1,299.50 )' -> Decimal('-1299.50')

== apply_discount ==
(Decimal('10.05'),10) -> 9.05   (Decimal('0.125'),0) -> 0.13   (Deci

### Assistant
The review passed with no counterexamples. Final confirmation:

### Tool call: execute
{"command": "cd workspace && python -m pytest tests -q 2>&1 | tail -3"}

### Tool result
......                                                                   [100%]
6 passed in 0.02s

[Command succeeded with exit code 0]

### Assistant
All 6 tests pass and an independent review found no counterexamples against the docstrings.

## Summary

The suite failed because three functions didn't implement what their docstrings promised. I changed only source files (nothing in `workspace/tests/`):

**`workspace/inventory/pricing.py`**
- `parse_price`: only stripped whitespace and `$`, so `"$1,299.50"` raised `decimal.InvalidOperation` and accounting-style negatives were unsupported. It now strips surrounding whitespace, treats a value wrapped in parentheses as negative (`"(12.00)"` → `-12.00`, `"($1,299.50)"` → `-1299.50`), removes `$` and `,` thousands separators, and raises a clear `ValueError` for unparseable or non-finite input instead of leaking `InvalidOperation`.
- `apply_discount`: `quantize(Decimal("0.01"))` used the default `ROUND_HALF_EVEN`, giving `9.04` for `Decimal("10.05")` at 10%. It now passes `rounding=ROUND_HALF_UP` (docstring's "round half up", `0.005 → 0.01`) and coerces int/float/str inputs to `Decimal` so the result is always a 2-decimal `Decimal`.

**`workspace/inventory/export.py`**
- `to_csv_row`: never quoted names, contradicting the RFC 4180 rule in its docstring. Names containing a comma or a double quote are now wrapped in double quotes with internal quotes doubled (`a,"b"` → `"a,""b"""`); price is formatted to exactly 2 decimals.

**`workspace/inventory/report.py`**
- `low_stock`: used `<=` (inclusive) and returned unsorted names, while the docstring says "strictly below" and "sorted alp