# Bug 3: Missing Async Function Extraction

**Severity:** Medium  
**Status:** Unfixed in v0.1.0  
**Location:** `_extract_python()` and `_extract_javascript()` in `plat_twin_maker.py`

## Description

The extraction functions miss `async def` in Python and `async () =>` / `async function()` in JavaScript. All async functions go undetected, meaning their PLATO tiles are never created.

## Root Cause

### Python

The check at line 240 only looks for `def `:

```python
if stripped.startswith('def '):
    match = re.match(r'def\s+(\w+)\s*\((.*?)\)(.*?):', stripped)
```

It does not check for `async def `. An async function like:

```python
async def fetch_data(url: str) -> dict:
    ...
```

is completely skipped.

### JavaScript

The regex at line 349 has two issues:

```python
for match in re.finditer(r'(?:export\s+)?(?:function|const|async function)\s+(\w+)', content):
```

1. **`async` is embedded in an alternation** — the regex matches `function`, `const`, or `async function`. This catches `async function foo()` but NOT `async foo =>` (async arrow) or `const foo = async () =>` (async arrow assigned to const)
2. **Full arrow functions are missed** — `const add = (a, b) => a + b` is not matched at all

## Reproduction

```python
# Python — will not be extracted
async def fetch_items(url: str) -> list[int]:
    response = await get(url)
    return response.json()

# JavaScript — missed patterns
const fetchData = async () => {
    const data = await api.get('/data');
    return data;
};

const process = async function(items) {
    return items.map(transform);
};

const add = (a, b) => a + b;  // Arrow function also missed
```

## Impact

Any Python codebase using `async def` (web frameworks, async I/O, data pipelines) will have incomplete PLATO twins. JavaScript/TypeScript codebases using modern patterns (arrow functions, async/await) will also miss significant portions of their module landscape.

## Fix

### Python
```python
if stripped.startswith('def ') or stripped.startswith('async def '):
    match = re.match(r'(?:async\s+)?def\s+(\w+)\s*\((.*?)\)(.*?):', stripped)
```

### JavaScript
```python
# Match named functions (regular + async)
for match in re.finditer(
    r'(?:export\s+)?(?:async\s+)?function\s+\*?\s*(\w+)', content
):
    ...

# Match const assignments (including arrow + async arrow)
for match in re.finditer(
    r'(?:export\s+)?(?:async\s+)?const\s+(\w+)\s*=\s*(?:async\s+)?(?:function\s*)?\(?[^)]*\)?\s*(?:=>)\s*(?:{|[\w])',
    content
):
    ...
```

Also consider using a proper AST parser (e.g., `ast` module for Python) instead of regex for both reliability and completeness.
