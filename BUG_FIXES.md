# Bug Fixes

This document summarizes the 3 confirmed bugs found during the v0.1.0 evaluation, with recommended fixes.

## Bug 1: Double-Arrow Signatures in Python Extraction

**Location:** `_extract_python()` in `plat_twin_maker.py`, lines 240-251

**Problem:** The regex `r'def\s+(\w+)\s*\((.*?)\)(.*?):'` uses a non-greedy `(.*?)` for the capture group that extracts return type annotations. When a function has a complex return type containing colons — for example `-> Callable[[int], str]` — the non-greedy match stops at the first `:` found inside the type annotation rather than the `:` at the end of the function signature.

**Example:**
```python
def process(items: list[int]) -> Callable[[int], str]:
```

The regex `(.*?):` captures ` -> Callable[[int]` instead of ` -> Callable[[int], str]:`. This produces the signature `def process(items: list[int]) ->  -> Callable[[int]` — note the double arrow.

**Fix:** Replace the non-greedy `(.*?):` with a regex that correctly handles nested brackets and parentheses before matching `:`:

```python
match = re.match(r'def\s+(\w+)\s*\((.*?)\)(\s*->\s*[^:]+)?:', stripped)
```

Or better, use a two-pass approach: match the full definition line, then split into `def name(args)` and everything before the final `:`.

**Status:** Unfixed in v0.1.0

---

## Bug 2: Language Detection Misclassification

**Location:** `_detect_language()` in `plat_twin_maker.py`, lines 151-162

**Problem:** The language detection uses a simple extension frequency count. For mixed-language repositories, the most common extension wins, which can misclassify the primary language. Specific issues:

1. **C vs C++ ambiguity** — `.h` files are mapped to `'c'`, so a C++ project with many `.h` headers gets classified as C
2. **JavaScript + TypeScript** — A TypeScript project with generated `.js` output files gets classified as JavaScript
3. **Configuration files** — `.json`, `.yaml`, `.toml`, `.md` files are counted equally with source code, potentially skewing results

**Example:** A C++ project with 15 header files and 10 `.cpp` files gets classified as `'c'` instead of `'cpp'`.

**Fix:** 
```python
def _detect_language(self) -> str:
    ext_counts = {}
    for f in self.path.rglob('*'):
        if f.is_file() and not any(p in str(f) for p in ['node_modules', '.git', '__pycache__', 'target', '.venv']):
            ext = f.suffix.lower()
            if ext in SUPPORTED_LANGUAGES:
                ext_counts[ext] = ext_counts.get(ext, 0) + 1
    if not ext_counts:
        return 'unknown'
    
    # Weight source-code extensions higher than config/data
    source_extensions = {'.py', '.rs', '.go', '.c', '.h', '.cpp', '.cc',
                         '.js', '.ts', '.jsx', '.tsx', '.java', '.rb',
                         '.php', '.swift', '.kt', '.scala', '.sh'}
    config_extensions = {'.md', '.json', '.yaml', '.yml', '.toml'}
    
    # Score languages
    lang_scores = {}
    for ext, count in ext_counts.items():
        lang = SUPPORTED_LANGUAGES[ext]
        weight = 3 if ext in source_extensions else 1
        lang_scores[lang] = lang_scores.get(lang, 0) + (count * weight)
    
    return max(lang_scores, key=lang_scores.get)
```

Additionally, `.h` files should be classified as C or C++ based on context (presence of `.cpp`/`.cc` files).

**Status:** Unfixed in v0.1.0

---

## Bug 3: Missing Async Function Extraction

**Location:** `_extract_python()`, `_extract_javascript()` in `plat_twin_maker.py`

**Problem:** The extraction functions miss async functions in both Python and JavaScript.

**Python:** The regex at line 240 starts with `def ` — `async def` functions are not matched:

```python
if stripped.startswith('def '):  # misses 'async def '
```

**JavaScript:** The regex at line 349 alternates `(?:function|const|async function)\s+(\w+)`. This misses:
- `const foo = async () => { ... }` — arrow functions with async
- `const foo = async function() { ... }` — async function expressions assigned to const
- Arrow functions in general: `const foo = () => { ... }`
- Short arrow functions: `const foo = x => x + 1`

**Fix for Python:**
```python
if stripped.startswith('def ') or stripped.startswith('async def '):
    match = re.match(r'(?:async\s+)?def\s+(\w+)\s*\((.*?)\)(.*?):', stripped)
```

**Fix for JavaScript:**
```python
# Match both regular and async functions, including arrow functions
for match in re.finditer(
    r'(?:export\s+)?(?:async\s+)?(?:function|const)\s+(\w+)', content
):
    # Also match arrow functions: const foo = (...) => ...
for match2 in re.finditer(
    r'(?:export\s+)?(?:async\s+)?const\s+(\w+)\s*=\s*(?:async\s+)?(?:\([^)]*\)|\w+)\s*=>', content
):
    ...
```

**Status:** Unfixed in v0.1.0
