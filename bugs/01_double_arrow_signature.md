# Bug 1: Double-Arrow Signatures

**Severity:** Medium  
**Status:** Unfixed in v0.1.0  
**Location:** `_extract_python()` in `plat_twin_maker.py`, lines 240-251

## Description

When `_extract_python()` parses function definitions with complex return type annotations containing colons — such as `Callable[[int], str]` — the regex produces an incorrect signature with a "double arrow": `-> ->`.

## Root Cause

The regex at line 241:

```python
match = re.match(r'def\s+(\w+)\s*\((.*?)\)(.*?):', stripped)
```

The third capture group `(.*?)` uses a **non-greedy** match for the return type annotation. It stops at the **first** `:` it encounters, not the final one that terminates the function signature. For example, given:

```python
def process(items: list[int]) -> Callable[[int], str]:
```

The regex captures:
- `(.*?)` = ` -> Callable[[int]` — stops at the colon inside `[int]` 

This produces the erroneous signature:
```
def process(items: list[int]) ->  -> Callable[[int]
```

The `-> Callable[[int]` (from return annotation) is followed by ` -> Callable[[int]` — a "double arrow."

## Reproduction

```python
from plato_twin_maker.plat_twin_maker import RepoAnalyzer

# Analyzer will extract this function
def process(items: list[int]) -> Callable[[int], str]:
    pass

# Expected signature: "def process(items: list[int]) -> Callable[[int], str]"
# Got: "def process(items: list[int]) ->  -> Callable[[int]"
```

## Fix

Replace the regex with one that correctly handles nested brackets:

```python
# Better regex: use a pattern that matches balanced brackets
match = re.match(r'def\s+(\w+)\s*\((.*?)\)(\s*->\s*(?:\[[^\]]*\]|[^:])+)?:', stripped)
```

Or use a two-pass approach:
1. Match `def name(args)` with the opening `(` and matching `)`
2. Everything between the closing `)` and the final unescaped `:` is the return annotation

## Impact

Affects any Python function with:
- `Callable` return types containing colons
- `dict[str, int]` return types
- TypedDict return annotations
- Any return type with nested `:` characters

Functions with simple return types (no `:` in the annotation) are unaffected.
