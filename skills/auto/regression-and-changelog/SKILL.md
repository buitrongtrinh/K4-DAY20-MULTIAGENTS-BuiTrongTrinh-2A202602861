---
name: regression-and-changelog
description: Use when a task requires fixing bugs and the conventions demand regression tests and a changelog entry for each fix.
---
1. For every bug fixed, add one test function to `tests/test_regressions.py` (at least the required minimum).
2. Ensure `tests/test_regressions.py` passes on its own.
3. Record each fix in `CHANGELOG.md` under the heading `## Unreleased` as a bullet `- fix(<function name>): <short description>`.
4. Add at least the required number of changelog bullets, one per fix.
5. Add type annotations to all parameters and return values of every public function (name not starting with `_`).
6. Self-check: (a) regression test file exists and passes, (b) changelog heading and bullet format match exactly, (c) bullet count meets the minimum, (d) all public functions are annotated.
