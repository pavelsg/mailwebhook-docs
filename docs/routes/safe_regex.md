---
title: Safe Regex
parent: Routes
nav_order: 5
---

# Safe regex
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Because unbounded regexes can hang worker processes, the routing
compiler lints every pattern and falls back to a
timeout-enforced executor . This page explains what gets
flagged, how evaluation works at runtime, and how to craft reliable patterns.

## Lint checks

Linting inspects each entry before compilation. Patterns that hit
any rule are logged and treated as *risky*, which
changes how they execute later.

Reasons surfaced today:

- Not a string (e.g., `123`).
- Longer than 500 characters.
- More than 80 capture groups.
- More than 100 alternations (`|`).
- Nested greedy quantifiers like `(.*)+` or `(\d+)+`.
- Backreferences (`\1`, `\2`, …).
- Lookbehind assertions (`(?<=...)`, `(?<!...)`).

The compiler still accepts risky expressions so existing routes keep working,
but warnings surface in the logs whenever lint fails.

## Execution path

Each rule compiles with `IGNORECASE` falg. Runtime behaves as follows:

1. **Normal patterns** – Lint passed and compilation succeeded. The engine runs
   regex search inline. Slow matches (>2× timeout) emit 
   logs but still return their result.
2. **Risky or failed compilation** – The engine delegates to
   `saf regex` wrapper, which executes the pattern in an isolated process with a
   deadline of 75 ms. If execution times out or
   raises, the match is treated as `False`.

## Best practices

- Keep expressions short (under 200 chars) and avoid nested wildcards. Anchors
  (`^`, `$`) generally improve both readability and speed.
- Prefer alternations over complex lookarounds. Lookbehind/backreferences force
  the engine into the risky path and may time out on realistic subjects.
- If you only need simple substring checks, use `subject_contains` instead—it
  never triggers regex safety rails and is easier to reason about.
