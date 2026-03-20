Install the OpenAI-compatible SDK:

pip install openai

Optional for plotting:

pip install matplotlib

Quickstart
1. Generate datasets

python scripts/generate_v1_benchmarks.py

2. Run a model on one benchmark

python scripts/run_model_openai_compatible.py \
  --dataset data/v1/constraint_stacking_test.jsonl \
  --output cs_preds.jsonl \
  --model route-llm \
  --api-key YOUR_API_KEY \
  --base-url https://routellm.abacus.ai/v1 \
  --temperature 0 \
  --max-tokens 32

3. Evaluate predictions

python scripts/evaluate_predictions.py \
  --dataset data/v1/constraint_stacking_test.jsonl \
  --predictions cs_preds.jsonl \
  --output cs_summary.json

4. Plot overload curves

python scripts/plot_v1_results.py --input cs_summary.json

Multi-model experiments
Run several models across several benchmarks:

python scripts/multi_model_orchestrator.py \
  --datasets \
    data/v1/constraint_stacking_test.jsonl \
    data/v1/extraneous_load_test.jsonl \
    data/v1/chunking_test.jsonl \
    data/v1/element_interactivity_test.jsonl \
  --models route-llm gpt-4o-mini \
  --api-key YOUR_API_KEY \
  --base-url https://routellm.abacus.ai/v1 \
  --temperature 0 \
  --max-tokens 32 \
  --output-dir runs/v1

Then compare results visually:

python scripts/comparison_plotter.py --input runs/v1/comparison.json --output-dir plots/v1

Metrics
Primary metrics:

Exact Accuracy
Mean Constraint Satisfaction
Condition-wise Accuracy by Load Level
Secondary analyses:

overload slope
chunked vs flat delta
low vs high interactivity gap
distractor sensitivity by distractor type
failure taxonomy frequencies
What counts as evidence for cognitive-load-like behavior?
We treat the following as evidence of structured overload:

exact accuracy drops faster than partial local success
distractor type changes performance more than raw length alone
chunking improves performance without changing content
high interactivity hurts more than equal-sized low-interactivity contexts
smaller models show steeper collapse curves
These patterns suggest bounded coordination capacity, not just generic noise.

Limitations
This is a functional analogy to cognitive load, not a claim of human-like subjective experience.
Synthetic tasks may under-represent natural language complexity.
Prompting and formatting choices can influence behavior.
Some errors may reflect training mismatch rather than load limits.
Future directions
paired-condition datasets
confidence and calibration analysis
logprob-based overload markers
architecture comparisons at fixed parameter budget
curriculum interventions for load robustness
hidden-state analyses of representational crowding
Citation-style summary
If you use or adapt this benchmark, cite it conceptually as:

A benchmark suite for studying structured overload behavior in tiny language models across constraint coordination, interference sensitivity, chunking benefit, and element interactivity.

License
Choose a license appropriate to your use case, e.g. MIT, Apache-2.0, or research-only.
