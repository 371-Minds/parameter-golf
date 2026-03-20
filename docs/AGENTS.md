# DEV.md

## Purpose

This file is the operating guide for AI agents and human collaborators working in this repository.

It exists to reduce drift, naming inconsistency, duplicate scripts, and unnecessary redesign.

This repository is a benchmark-and-analysis system for testing whether LLMs exhibit overload-like failure patterns analogous to cognitive load.

AI agents working in this repo should optimize for:

- clarity
- consistency
- reproducibility
- minimal surprise
- backward-compatible extension where possible

## Core repository philosophy

This is not a generic ML repo.

It is a **research scaffold** built around a specific thesis:

> LLMs may exhibit bounded-capacity processing behavior that can be analyzed through the lens of cognitive load theory.

The repository should therefore preserve:

- benchmark clarity
- causal interpretability where possible
- matched-item design in v2
- legible statistical outputs
- reproducible script interfaces

## Canonical directories

Use these directories unless explicitly instructed otherwise:

- `data/v1/`
- `data/v2/`
- `scripts/`
- `runs/v1/`
- `paired_v2_runs/`
- `plots/v1/`
- `plots/v2/`
- `forest_plots_corrected/`

Do not invent new top-level directories unless there is a strong reason.

## Canonical filenames

Use these names exactly for existing functionality:

### Root files
- `README.md`
- `AGENTS.md`
- `DEV.md`

### Data generation
- `scripts/generate_v1_benchmarks.py`
- `scripts/generate_v2_benchmarks.py`

### Evaluation and plotting
- `scripts/evaluate_predictions.py`
- `scripts/plot_v1_results.py`

### Running
- `scripts/run_model_openai_compatible.py`

### V1 orchestration
- `scripts/multi_model_orchestrator.py`
- `scripts/comparison_plotter.py`

### V2 orchestration
- `scripts/paired_orchestrate_v2.py`
- `scripts/paired_comparison_plotter.py`

### Statistical testing
- `scripts/cross_model_significance.py`
- `scripts/cross_model_permutation.py`
- `scripts/cross_model_permutation_corrected.py`

### Corrected visualization
- `scripts/forest_plot_corrected.py`

If functionality overlaps one of these files, extend the existing file or create a clearly versioned successor.  
Do not create near-duplicates with vague names.

## Naming conventions

### Scripts
Use `snake_case.py`.

Prefer names that reveal role, such as:

- `generate_*`
- `evaluate_*`
- `plot_*`
- `orchestrate_*`
- `compare_*`
- `cross_model_*`

Avoid ambiguous names like:
- `final.py`
- `new_script.py`
- `analysis2.py`
- `temp_runner.py`

### Data files
Use descriptive benchmark names:

- `constraint_stacking_train.jsonl`
- `chunking_v2_test.jsonl`

### Prediction files
Use model and experiment in the filename.

Preferred pattern:

`{model_name}__{experiment_name}__predictions.jsonl`

### Paired results
Preferred pattern:

`{model_name}__{experiment_name}__paired_results.jsonl`

### Summary files
Preferred pattern:

`{model_name}__{experiment_name}__summary.json`

### Plot files
Use the experiment and plot purpose in the filename.

Examples:
- `chunking_v2__forest_bh.png`
- `extraneous_gap_irrelevant_minus_confusable.png`

## Workflow hierarchy

AI agents should preserve this conceptual pipeline:

### Stage 1: dataset generation
Create benchmark JSONL files with stable schema.

### Stage 2: model execution
Run model inference against datasets and save raw predictions.

### Stage 3: evaluation
Compare predictions against gold targets and compute metrics.

### Stage 4: aggregation
Summarize results across models, conditions, or matched pairs.

### Stage 5: statistical testing
Estimate uncertainty and model differences using:
- bootstrap CI
- permutation tests
- multiple-comparison correction where relevant

### Stage 6: visualization
Produce plots that expose:
- effect size
- direction
- uncertainty
- corrected significance

Do not collapse multiple stages into a single opaque script unless explicitly asked.

## Benchmark design rules

When creating or extending benchmarks, preserve the core conceptual distinctions.

### Constraint stacking
Manipulate number of active constraints while minimizing unrelated variation.

### Extraneous load
Manipulate distractor type, not just distractor quantity.

Preferred distractor hierarchy:
- irrelevant
- confusable
- contradictory or contradiction-adjacent

### Chunking
Keep semantic content constant while changing presentation structure.

### Element interactivity
Manipulate dependency coupling while holding superficial length as stable as possible.

### Paired v2 design
When possible, use matched-item design:
- same latent task
- multiple variants
- shared `pair_id`

This is strongly preferred over unrelated-condition comparisons.

## Schema rules

Benchmark rows should remain JSONL and use stable keys.

Preferred fields:

- `id`
- `experiment`
- `prompt`
- `target`
- `grader`

For paired benchmarks also include:

- `pair_id`
- `variant`

Optional but useful:

- `base_item_id`
- `condition`
- `metadata`

Do not casually rename schema keys once downstream scripts depend on them.

If schema changes are necessary:
- update all dependent scripts
- update README
- document the change in comments or commit notes

## Grading rules

Prefer simple, explicit grading where possible.

Default:
- exact normalized match

Allowed extensions:
- accepted-answer lists
- constrained symbolic graders
- lightweight rule-based graders

Avoid introducing overly magical grading logic without documentation.

If free-form grading is added later, it must be:
- documented
- testable
- reproducible

## Statistical rules

When comparing models on paired effects:

- prefer within-item paired effect estimates
- use bootstrap confidence intervals for effect uncertainty
- use sign-flip permutation tests for null comparison where appropriate
- apply multiple-comparison correction if running many tests

Supported correction methods in this repo:
- BH-FDR
- Holm
- Bonferroni

Default recommendation:
- use BH-FDR for exploratory analysis
- use Holm or Bonferroni for stricter confirmatory claims

## Visualization rules

Plots should answer a research question clearly.

Preferred plot types:
- condition comparison bars
- paired effect summaries
- model comparison plots
- corrected forest plots

Every plot should make at least one of these visible:
- effect magnitude
- effect direction
- uncertainty
- corrected significance
- benchmark-specific overload pattern

Avoid decorative plots with low interpretive value.

## CLI design rules

Scripts should use stable command-line interfaces.

Preferred conventions:
- `--input`
- `--inputs`
- `--output`
- `--output-dir`
- `--json-out`
- `--md-out`
- `--model`
- `--models`
- `--dataset`
- `--datasets`
- `--base-url`
- `--api-key`
- `--seed`
- `--n-perm`
- `--alpha`

Do not create unnecessary argument naming variations if an established pattern already exists.

## AI-agent implementation advice

### 1. Preserve contracts
Before modifying a script, identify:
- expected inputs
- expected outputs
- filename conventions
- schema assumptions

Do not break them casually.

### 2. Prefer extension over duplication
If a script already exists for a function, extend it rather than creating:
- `*_v3.py`
- `*_new.py`
- `*_fixed.py`

unless versioning is truly necessary.

### 3. Be explicit
When writing code:
- avoid hidden assumptions
- avoid implicit schema inference if direct fields exist
- prefer readable code over clever code

### 4. Keep outputs inspectable
Write JSON and markdown summaries wherever useful.

Good outputs:
- machine-readable JSON
- human-readable markdown
- clearly named plot files

### 5. Optimize for human review
This repo is AI-assisted and may be reviewed by nontraditional builders.

Favor:
- readable function names
- short helper functions
- obvious file outputs
- comments at conceptual boundaries

### 6. Avoid fake sophistication
Do not add complexity just to appear advanced.

Examples to avoid:
- unnecessary class hierarchies
- overengineered abstractions
- hidden side effects
- implicit caching without documentation

### 7. Treat this as research infrastructure
Every addition should answer:
- what research question does this serve?
- what existing output does this connect to?
- how will a human interpret the result?

If the answer is unclear, reconsider the change.

## Common mistakes to avoid

Avoid these failure modes:

- introducing duplicate scripts for nearly identical tasks
- renaming files without updating README
- changing JSON schema without updating evaluators
- adding plots that don’t map to a benchmark hypothesis
- mixing v1 and v2 outputs in the same directory without labeling
- using inconsistent model-name normalization in filenames
- writing outputs whose purpose is unclear from the filename

## Definition of done

A new script or modification is complete only if:

1. the filename is consistent
2. inputs and outputs are clear
3. output files are saved to expected directories
4. README references are updated if needed
5. the code is inspectable and not overly magical
6. the change serves a real benchmark or analysis purpose

## Hints for future AI agents

### If extending the benchmark suite
Prefer:
- paired designs
- stronger controls
- easier causal interpretation
- stable schema

### If extending statistics
Prefer:
- transparent effect definitions
- paired tests when possible
- correction for multiple comparisons
- human-readable summaries

### If extending plotting
Prefer plots that make uncertainty visible.

### If extending model running
Preserve OpenAI-compatible API support first.
Only add other backends if they are clearly separated and documented.

### If extending documentation
Keep documentation honest:
- do not overclaim
- distinguish hypothesis from proof
- distinguish scaffold from validated result

## Author context

This repository was built in an AI-assisted way by a nontraditional builder.

That means future agents should optimize for:
- operational clarity
- explainability
- handoff quality
- conceptual fidelity

not for:
- showing off programming cleverness
- maximal abstraction
- unnecessary reinvention

## Final rule

If a future AI agent is unsure what to optimize for, optimize for this:

**Make the repository easier to understand, easier to run, and more faithful to the core research thesis.**
