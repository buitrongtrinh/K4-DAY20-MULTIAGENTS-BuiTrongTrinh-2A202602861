---
name: follow-stated-conventions
description: Use when a task specifies explicit output conventions such as file names, JSON keys, headings, formats, or naming rules that must be matched exactly.
---
1. Extract every explicit convention from the task/feedback into a checklist before writing code.
2. Match names and formats exactly as stated: file paths, JSON keys, heading text, bullet formats, casing, separators.
3. Apply stated transformations literally (e.g. lower-case and replace '-' with '_' in names; sort by the stated keys in the stated direction).
4. When a docstring or spec describes supported formats or ordering, implement all listed cases, not just the common one.
5. For numeric/money outputs, follow the stated representation (e.g. integer cents) and rounding rule exactly.
6. Self-check: (a) each convention is implemented, (b) a sample output matches the stated format character-for-character, (c) edge cases named in the spec are handled.
