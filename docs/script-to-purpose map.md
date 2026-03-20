# Script Purpose Map

## Dataset generation
- `generate_v1_benchmarks.py`  
  Generates non-paired synthetic benchmarks:
  - constraint stacking
  - extraneous load
  - chunking
  - element interactivity

- `generate_v2_benchmarks.py`  
  Generates paired matched-item benchmarks:
  - chunking_v2
  - extraneous_load_v2
  - element_interactivity_v2

## Model running
- `run_model_openai_compatible.py`  
  Runs one model on one dataset using an OpenAI-compatible API.

## Single-dataset evaluation
- `evaluate_predictions.py`  
  Evaluates predictions against benchmark JSONL and writes summary metrics.

## V1 orchestration and plotting
- `multi_model_orchestrator.py`  
  Runs multiple models across v1 datasets and writes aggregate comparison output.

- `comparison_plotter.py`  
  Plots cross-model v1 benchmark results from `comparison.json`.

- `plot_v1_results.py`  
  Plots single-run or condition-level v1 summaries.

## V2 paired orchestration and plotting
- `paired_orchestrate_v2.py`  
  Runs multiple models across paired v2 datasets and computes paired effects.

- `paired_comparison_plotter.py`  
  Visualizes model-level paired effects from `paired_comparison.json`.

## Cross-model statistical testing
- `cross_model_significance.py`  
  Bootstrap CI based cross-model comparison on paired effects.

- `cross_model_permutation.py`  
  Sign-flip permutation testing for cross-model comparison on paired effects.

- `cross_model_permutation_corrected.py`  
  Same as above, but adds multiple-comparison correction:
  - BH-FDR
  - Holm
  - Bonferroni

## Corrected plotting
- `forest_plot_corrected.py`  
  Forest plots for corrected cross-model paired-effect comparisons.
