# Bug 2: Language Detection Misclassification

**Severity:** Low  
**Status:** Unfixed in v0.1.0  
**Location:** `_detect_language()` in `plat_twin_maker.py`, lines 151-162

## Description

The primary language detection uses a simple extension frequency count without weighting. This leads to misclassification of mixed-language repositories.

## Root Cause

The `_detect_language` method counts all supported file extensions equally:

```python
ext_counts = {}
for f in self.path.rglob('*'):
    if f.is_file() and not any(p in str(f) for p in ['node_modules', '.git', ...]):
        ext = f.suffix.lower()
        if ext in SUPPORTED_LANGUAGES:
            ext_counts[ext] = ext_counts.get(ext, 0) + 1
return SUPPORTED_LANGUAGES.get(max(ext_counts, key=ext_counts.get), 'unknown')
```

Problems:

1. **`.h` → `'c'` mapping**: A C++ project with `.h` headers gets classified as C because `.h` maps to `'c'`
2. **Config files skew results**: A TypeScript project with many `.json` config files could get misclassified as JSON
3. **Build artifacts**: Generated/built `.js` files in a TypeScript repo inflate JavaScript counts
4. **No threshold**: Even a single file of the wrong type can be the plurality winner if distribution is even

## Reproduction

```python
# C++ project with 15 .h files and 10 .cpp files
# → 15 "c" votes vs 10 "cpp" votes
# → Primary language: 'c' (WRONG)

# TypeScript project with 20 .ts files and 8 generated .js files
# → 20 typescript votes vs 8 javascript votes → correct by count
# But with 50 .json config files → 22+50 split → could skew
```

## Fix

Replace the simple count with a weighted scoring system:

```python
SOURCE_WEIGHT = 3
CONFIG_WEIGHT = 1

source_exts = {'.py', '.rs', '.go', '.c', '.h', '.cpp', '.cc',
               '.js', '.ts', '.jsx', '.tsx', '.java', '.rb',
               '.php', '.swift', '.kt', '.scala', '.sh', '.bash'}

# Additionally, if both .cpp and .h exist, classify .h as C++
if ext == '.h' and any(f.suffix in {'.cpp', '.cc'} for f in ...):
    lang = 'cpp'
```

## Impact

- Mixed C/C++ repos get wrong primary language → wrong room naming, wrong tags
- Generated `.js` in TypeScript repos dilutes accuracy
- Config-heavy repos (JSON/YAML/TOML) may not be detected as their actual source language
