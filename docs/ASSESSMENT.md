# Docs Assessment & Build-Readiness Review

**Date:** 2026-03-20
**Scope:** Full review of all 30+ documents in `docs/`, cross-referenced against the actual repository state.

---

## Confidence Score

### Overall: **72 / 100 — High-quality specs, actionable gaps need fixing before build**

| Area | Score | Notes |
|------|-------|-------|
| Benchmark design quality | 92 | Exceptionally well-specified; 4 V1 + 3 V2 benchmarks with clear generation logic |
| Statistical analysis design | 88 | Solid pipeline: bootstrap CI → permutation tests → multiple-comparison correction |
| Governance & conventions | 85 | AGENTS.md is thorough; naming, schema, and CLI conventions are clear |
| Script specifications | 80 | Every script has a spec doc; CLI interfaces and I/O contracts are defined |
| Internal consistency | 55 | Naming mismatches between Installation.md, README.md, and canonical map |
| Build readiness | 60 | Specs are implementable but nothing exists yet — no `scripts/` dir, no `data/v1/` |
| Documentation completeness | 70 | Some docs are raw chat exports, not polished specs (Teardown.md, Experiments to test.md, README-ai.md) |

---

## Executive Summary

The docs folder contains a **comprehensive, well-designed research framework** for testing cognitive-load-like failure patterns in language models. The benchmark specifications, evaluation harness, statistical pipeline, and visualization specs are all strong enough to build from directly.

However, there are **three categories of issues** that should be resolved before starting implementation:

1. **Naming inconsistencies** across documents that would cause confusion during build
2. **Raw chat exports** mixed in with polished specs that need cleanup or reclassification
3. **Missing structural prerequisites** (no `scripts/` or `data/v1/` directories exist yet)

None of these are blockers — they are fixable in a single session. Once resolved, the specs are ready for implementation.

---

## What's Strong

### 1. Benchmark Design (Synthetic dataset spec.md + Concrete experimental matrix.md)
- Four V1 benchmarks (constraint stacking, extraneous load, chunking, element interactivity) are fully specified with generation algorithms, difficulty knobs, example JSONL, and anti-cheat controls.
- Three V2 paired benchmarks (chunking_v2, extraneous_load_v2, element_interactivity_v2) define matched-item designs with explicit paired-effect formulas.
- The experimental matrix defines 7 experiments with model sizes, metrics, error taxonomy, and statistical analysis plan.

### 2. Evaluation & Statistical Pipeline
- **Evaluation harness.md** provides complete Python code for exact-match and constraint-checker grading with per-condition breakdowns.
- **Statistical testing script.md** → **cross-model significance script.md** → **permutation test version.md** → **multiple comparison correction.md** form a clean progression from basic to rigorous analysis.
- Correction methods (BH-FDR, Holm, Bonferroni) are specified with clear use-case guidance.

### 3. Governance (AGENTS.md)
- Canonical directories, filenames, naming conventions, CLI patterns, schema rules, grading rules, and workflow hierarchy are all documented.
- The "definition of done" and "common mistakes to avoid" sections are practically useful for any implementer.

### 4. Script-to-Purpose Mapping
- **Canonical repo file map.md** and **script-to-purpose map.md** together define exactly 15 scripts with clear roles and stable filenames.
- Every script has a corresponding spec document with CLI interface and I/O contracts.

---

## Issues Found

### Issue 1: Naming Inconsistencies (HIGH — fix before build)

**Installation.md** uses different script names than the canonical map:

| Installation.md says | Canonical map says |
|-----|-----|
| `generate_datasets.py` | `scripts/generate_v1_benchmarks.py` |
| `run_benchmark.py` | `scripts/run_model_openai_compatible.py` |
| `evaluate.py` | `scripts/evaluate_predictions.py` |
| `plot_overload.py` | `scripts/plot_v1_results.py` |
| `orchestrate_benchmarks.py` | `scripts/multi_model_orchestrator.py` |
| `plot_comparison.py` | `scripts/comparison_plotter.py` |

**Impact:** Anyone following Installation.md will create wrong filenames and break the pipeline.

**Recommendation:** Update Installation.md to use canonical names with `scripts/` prefix.

### Issue 2: Directory Structure Mismatch (HIGH — fix before build)

**docs/README.md** shows a repository structure with data files at `data/` root level:
```
data/
├── constraint_stacking_train.jsonl
├── constraint_stacking_test.jsonl
```

**Canonical map** and **AGENTS.md** specify:
```
data/v1/
data/v2/
```

**Additionally**, docs/README.md references `scripts/generate_benchmarks.py` (singular) but the canonical map specifies both `generate_v1_benchmarks.py` and `generate_v2_benchmarks.py`.

**Impact:** Ambiguous directory structure will cause scripts to disagree on where to read/write data.

**Recommendation:** Align docs/README.md structure diagram with the canonical map.

### Issue 3: Raw Chat Exports Posing as Specs (MEDIUM — reclassify)

Several documents are clearly **raw AI chat outputs** rather than polished specifications:

| Document | Evidence |
|----------|----------|
| **Teardown.md** | Conversational tone ("Your intuition is strong"), second-person address, ends with "If you want, I can go one step deeper" |
| **Experiments to test.md** | Same pattern — "If you want, I can next turn this into..." |
| **README-ai.md** | Framing advice ("That makes you look modern, not apologetic"), mixed spec + coaching |
| **HAI.com** | Appears to be a raw attribution/transparency statement |

**Impact:** These documents contain valuable conceptual content but their conversational format makes it harder to extract actionable specs. They also mix strategic advice with technical specifications.

**Recommendation:** Keep these as reference material but distinguish them from actionable specs. Consider adding a `docs/reference/` subdirectory or a note at the top of each file indicating its status (e.g., "Reference — conceptual background" vs. "Spec — ready to implement").

### Issue 4: Overlapping Experiment Definitions (MEDIUM)

Three documents define experiments with different scopes and numbering:

| Document | Experiments | Status |
|----------|-------------|--------|
| **Experiments to test.md** | 10 experiments (E1–E10) | Broad exploratory list from chat |
| **Concrete experimental matrix.md** | 7 experiments (E1–E7) | Focused research design |
| **Synthetic dataset spec.md** | 4 benchmarks | Generation-ready specs |

The four generation-ready benchmarks (constraint stacking, extraneous load, chunking, element interactivity) are a subset of the experimental matrix, which is a subset of the exploratory list.

**Impact:** An implementer needs to know which experiments to build first. The answer is the 4 in Synthetic dataset spec.md, but this isn't stated explicitly.

**Recommendation:** Add a note to Concrete experimental matrix.md and Experiments to test.md clarifying that the Synthetic dataset spec.md defines the Tier 1 build targets.

### Issue 5: Dual Project Identity (LOW — by design, but worth noting)

The repository serves two purposes:
1. **Parameter golf challenge** — training tiny LLMs on FineWeb (fully implemented, 10+ submissions)
2. **Cognitive load benchmarks** — testing LLM overload patterns (fully specified, nothing implemented)

The root README.md is about parameter golf. The docs/README.md is about cognitive load benchmarks. The Teardown.md bridges them conceptually but operationally they are separate workstreams.

**Impact:** A new contributor might be confused about which part they're building.

**Recommendation:** This dual identity is intentional and well-managed. No change needed, but consider adding a one-line note to the root README.md acknowledging the cognitive load research framework in `docs/`.

### Issue 6: JSONL datasets.md Contains Inline Code (LOW)

**JSONL datasets.md** contains a complete Python script for generating all 4 benchmarks. This code should eventually live in `scripts/generate_v1_benchmarks.py`, not in a markdown file.

**Impact:** None currently, but the code will need to be extracted during implementation.

**Recommendation:** During build, extract the code from JSONL datasets.md into `scripts/generate_v1_benchmarks.py` rather than rewriting from scratch.

---

## Pre-Build Checklist

Before starting implementation, resolve these items:

- [ ] **Fix Installation.md** — Update all script names to match canonical map (with `scripts/` prefix)
- [ ] **Fix docs/README.md** — Align directory structure diagram with canonical `data/v1/`, `data/v2/` layout; fix script name references
- [ ] **Create directory skeleton** — `scripts/`, `data/v1/`, `data/v2/`, `runs/v1/`, `paired_v2_runs/`, `plots/v1/`, `plots/v2/`, `forest_plots_corrected/`
- [ ] **Add .gitkeep files** to empty directories so they persist in git
- [ ] **Add header notes** to raw chat exports (Teardown.md, Experiments to test.md, README-ai.md) marking them as reference material vs. actionable specs

---

## Recommended Build Order

Based on the spec quality and dependency chain:

### Phase 1: Core Pipeline (start here)
1. `scripts/generate_v1_benchmarks.py` — extract from JSONL datasets.md
2. `scripts/evaluate_predictions.py` — extract from Evaluation harness.md
3. `scripts/run_model_openai_compatible.py` — from Model runner script.md
4. `scripts/plot_v1_results.py` — from Plotting script.md

### Phase 2: Multi-Model Orchestration
5. `scripts/multi_model_orchestrator.py` — from Multi-model experiment orchestrator.md
6. `scripts/comparison_plotter.py` — from Comparison plotter for comparison.json.md

### Phase 3: Paired V2 Benchmarks
7. `scripts/generate_v2_benchmarks.py` — from Paired-condition benchmark v2 generator.md
8. `scripts/paired_orchestrate_v2.py` — from Multi-model paired orchestrator for v2.md
9. `scripts/paired_comparison_plotter.py` — from Paired comparison plotter.md

### Phase 4: Statistical Rigor
10. `scripts/cross_model_significance.py` — from cross-model significance script.md
11. `scripts/cross_model_permutation.py` — from permutation test version.md
12. `scripts/cross_model_permutation_corrected.py` — from multiple comparison correction.md
13. `scripts/forest_plot_corrected.py` — from forest plot script for corrected comparisons.md

### Phase 5: Documentation & Polish
14. `scripts/markdown_results_compiler.py` — from Markdown results compiler.md (if needed)
15. Update all READMEs with working examples
16. End-to-end pipeline validation

---

## What's Missing That You'll Need

| Need | Source | Priority |
|------|--------|----------|
| OpenAI API key or compatible endpoint | User-provided | Required for Phase 1 Step 3+ |
| `matplotlib` + `numpy` | `pip install` | Required for plotting scripts |
| `openai` Python SDK | `pip install` | Required for model runner |
| Target models for testing | User decision | Required before Phase 2 |
| Decision on V2 paired benchmark scope | Per Paired-condition benchmark v2 generator.md | Required before Phase 3 |

---

## Documents Inventory with Status

| Document | Type | Build-Ready? | Notes |
|----------|------|:---:|-------|
| AGENTS.md | Governance | ✅ | Strong, no changes needed |
| Canonical repo file map.md | Contract | ✅ | Source of truth for filenames |
| script-to-purpose map.md | Index | ✅ | Consistent with canonical map |
| Data format.md | Schema spec | ✅ | Clear JSONL schema |
| Synthetic dataset spec.md | Generation spec | ✅ | Complete generation algorithms for 4 benchmarks |
| JSONL datasets.md | Code + spec | ✅ | Contains extractable Python code |
| Evaluation harness.md | Code + spec | ✅ | Contains extractable evaluation code |
| Model runner script.md | Script spec | ✅ | Clear I/O contract |
| Multi-model experiment orchestrator.md | Script spec | ✅ | Clear coordination logic |
| Comparison plotter for comparison.json.md | Script spec | ✅ | Clear plot spec |
| Plotting script.md | Script spec | ✅ | Clear plot spec |
| Paired-condition benchmark v2 generator.md | Script spec | ✅ | Clear paired-item generation |
| Multi-model paired orchestrator for v2.md | Script spec | ✅ | Clear orchestration logic |
| Paired comparison plotter.md | Script spec | ✅ | Clear plot spec |
| Paired evaluator and paired stats pipeline.md | Script spec | ✅ | Clear evaluation pipeline |
| Statistical testing script.md | Script spec | ✅ | Bootstrap CI implementation |
| cross-model significance script.md | Script spec | ✅ | Cross-model comparison |
| permutation test version.md | Script spec | ✅ | Sign-flip permutation tests |
| multiple comparison correction.md | Script spec | ✅ | BH-FDR, Holm, Bonferroni |
| forest plot script for corrected comparisons.md | Script spec | ✅ | Corrected forest plots |
| Concrete experimental matrix.md | Research design | ✅ | 7-experiment matrix |
| Experimental results template.md | Template | ✅ | Result reporting structure |
| Research memo skeleton.md | Template | ✅ | Write-up structure |
| Markdown results compiler.md | Script spec | ✅ | Results → markdown |
| README.md (docs/) | Overview | ⚠️ | Directory structure needs alignment |
| Installation.md | Quickstart | ⚠️ | Script names need fixing |
| Teardown.md | Reference | ℹ️ | Raw chat export — valuable but not a spec |
| Experiments to test.md | Reference | ℹ️ | Raw chat export — exploratory experiment list |
| README-ai.md | Reference | ℹ️ | Raw chat export — README draft + coaching |
| HAI.com | Reference | ℹ️ | AI attribution/transparency statement |

**Legend:** ✅ = Ready to build from · ⚠️ = Needs fixes first · ℹ️ = Reference only, not a build spec

---

## Bottom Line

The specs are **strong enough to build a complete cognitive load benchmark suite**. The benchmark design is well-thought-out, the statistical pipeline is rigorous, and the governance conventions are clear.

The main risk is **naming drift** — if you start building without first aligning Installation.md and docs/README.md with the canonical map, you'll create inconsistencies that compound over time.

**Fix the naming issues first, then build in the recommended phase order, and the specs will carry you through.**
