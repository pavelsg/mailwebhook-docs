---
title: Rules
parent: Routes
nav_order: 1
---

# Rules
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Routes only run a transform pipeline when their routing rule matches the incoming
message. Rules are stored as arbitrary JSON, validated, and compiled into predicates in. 
The notes below explain what data is available to a rule,
how each field behaves, and how to combine matchers.

## What a rule sees

Before matching, the inbound email is normalized into a dictionary:

- `subject`: lowercased string.
- `to_emails`, `from_emails`: lists of lowercased email addresses. Empty when
  the parsed message omitted the field.
- `from_domains`: the domain portion (after the `@`) for every `from_emails`
  entry that contains a domain.
- `headers`: map of header name unfolded string, with header names lowercased
  and missing values coerced to `""`.

Every string comparison performed by the compiler lowercases both sides, so the
rule JSON can use any casing. Leading/trailing whitespace is stripped by
rule validators. Unknown keys are accepted but ignored by the compiler,
allowing forward-compatible storage.

## Leaf matchers

Each populated field is AND-ed at the top level (a message must satisfy all of
them unless you wrap logic in `any`/`all`/`none`/`negate`). Empty arrays or
objects mean “no constraint” for that field.

| Key | Type | Behavior |
| --- | --- | --- |
| `subject_contains` | `list[str]` | Passes when **any** entry is a substring of the subject. Comparisons are case-insensitive and operate on the normalized subject. |
| `subject_regex` | `list[str]` | Python regexes tested (case-insensitive) against the subject. Patterns are linted via `safe_regex`; invalid expressions are skipped and logged. |
| `to_contains` / `from_contains` | `list[str]` | Substring match against each recipient/sender email. If any address contains any substring, the predicate passes. |
| `to_emails` / `from_emails` | `list[str]` | Exact (case-insensitive) match against normalized addresses. |
| `from_domains` | `list[str]` | Exact match against any normalized sender domain. |
| `headers_equals` | `dict[str,str]` | Header must exist and exactly equal the provided value (name comparison is case-insensitive). |
| `headers_contains` | `dict[str,str]` | Header must include the provided substring. Useful for prefixes like `"x-priority": "high"`. |

## Boolean combinators

Rules can branch using nested boolean structures—each child entry uses the same
schema as a normal rule:

- `all`: list of subrules. Equivalent to logical AND.
- `any`: list of subrules. Equivalent to logical OR; an empty list evaluates to
  `True`, so omit the key (instead of sending `[]`) when you want “no constraint.”
- `none`: list of subrules. Passes only if none of the child subrules match.
- `negate`: single subrule whose result is inverted.

These combinators can be nested arbitrarily, so constructs like “match invoices
but not auto-replies” or “subject contains term *and* headers mention Plan X”
are expressible.

## Example rule JSON

```json
{
  "to_emails": ["alerts@example.com"],
  "from_domains": ["vendor.com"],
  "subject_contains": ["invoice", "receipt"],
  "any": [
    { "headers_contains": { "x-priority": "high" } },
    { "subject_regex": ["(?i)urgent"] }
  ],
  "none": [
    { "from_emails": ["bot@vendor.com"] }
  ]
}
```

With this rule saved on a route, only messages addressed to `alerts@example.com`
and sent by `vendor.com` will reach the route’s transform [pipeline]. 
The `any` branch makes high-priority or “urgent” subjects
eligible, while the `none` branch blocks a known autoresponder address.

## Operational notes

- Storing a rule with zero populated fields yields a “match everything” route,
  which is useful when you only rely on downstream mappers.
- Regex patterns run through [`safe_regex`], so catastrophic backtracking
  is mitigated; nevertheless, keep expressions narrow and test them locally.
- The UI and API return rules exactly as saved (`RouteOut.rule`), so you can
  clone an existing route by copying its JSON block verbatim.

[Pipeline]: {% link docs/routes/pipeline.md %}
[`safe_regex`]: {% link docs/routes/safe_regex.md %}