# Improvement Roadmap

Prioritized recommendations for plato-twin-maker from v0.1.1 hotfix through v0.3.0 production readiness.

---

## v0.1.1 — Hotfix & Stability

**Focus:** Fix confirmed bugs and harden the core pipeline.

| # | Improvement | Rationale |
|---|-------------|-----------|
| 1 | **Fix double-arrow signatures** | Regex fix for Python return type extraction with nested types |
| 2 | **Fix async function extraction** | Add `async def` support in Python and `async () =>` in JavaScript |
| 3 | **Fix language detection weights** | Source-code extensions should outrank config/data files; C++ detection from `.h` context |
| 4 | **Improve error handling** | Wrap extraction in try/except to handle malformed files without crashing the entire twin creation |

**Estimated effort:** 1-2 days

---

## v0.2.0 — Language & Features

**Focus:** Expand language coverage, export formats, and entry point detection.

| # | Improvement | Rationale |
|---|-------------|-----------|
| 5 | **Add more language extractors** | Currently supports 7 languages. Add Dart, Kotlin, Swift, Ruby, PHP with proper regex extractors |
| 6 | **Support multiple output formats** | JSON manifest, YAML, and markdown summaries in addition to PLATO tile submission |
| 7 | **Tune entry point scanning** | Add language-specific heuristics (e.g., `package.json` `"main"` field, Cargo.toml `[[bin]]`, `main.rs`) |
| 8 | **Add dependency graph export** | Generate Mermaid.js dependency graphs from extracted module relationships |

**Estimated effort:** 1 week

---

## v0.3.0 — Scale & PLATO Native

**Focus:** Batch processing, streaming, and production CLI.

| # | Improvement | Rationale |
|---|-------------|-----------|
| 9 | **Batch multi-repo processing** | Accept a list of repos for batch twin creation, with progress bars and summary reports |
| 10 | **Streaming tile submission** | Submit tiles to PLATO incrementally as they're created, rather than all-at-once at the end |
| 11 | **Configurable confidence thresholds** | Allow users to set minimum confidence via `--min-confidence 0.8` to skip low-quality extractions |
| 12 | **CLI refinements** | Add `--output-dir`, `--tags` (extra tags), `--exclude` patterns, `--max-files` limit, and `--quiet` mode |

**Estimated effort:** 2 weeks

---

## Additional Considerations

### Documentation
- Add docstrings to all extraction methods
- Provide a troubleshooting section for common issues (PLATO unreachable, unsupported languages, large repos)
- Add a CONTRIBUTING guide for adding new language extractors

### Testing
- Add property-based tests for signature extraction (Hypothesis)
- Add integration tests with real repos (small, sanitized fixtures)
- Add regression tests for each reported bug

### Performance
- Parallelize file processing with `concurrent.futures`
- Add a file-size limit option (`--max-file-size 1MB`)
- Cache regex compilations

### Security
- Sanitize repo paths to prevent path traversal
- Add timeout for git clone operations
- Validate PLATO URLs before submission attempts
