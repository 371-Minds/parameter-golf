Cross-model significance script

Yes — this is the statistically serious next layer.

This script compares paired effect sizes across models using the detailed paired result files, not just the aggregated paired_comparison.json.

That matters because to compare models properly, you want both models evaluated on the same paired items.

What this script does

Given paired result JSONL files for multiple models, it computes:

within-model paired effects
cross-model differences in those effects
bootstrap confidence intervals for model-vs-model effect differences

It supports:

chunking_v2

Compare models on:

Δchunk(m)​=E[chunked correct−flat correct]

Then compare two models:

Δchunk(A−B)​=Δchunk(A)​−Δchunk(B)​
element_interactivity_v2

Compare:

Δinteractivity(m)​=E[low correct−high correct]
extraneous_load_v2

Compare:

irrelevant - confusable
irrelevant - contradictory_adjacent
confusable - contradictory_adjacent
Why this is strong

It controls for:

same benchmark items
same pair structure
same latent examples

So the comparison becomes:

which model is more sensitive to chunking?
which model is more harmed by interactivity?
which model is more vulnerable to confusable distractors?
Input format

You pass one or more detailed paired result files of the kind produced by:

paired_evaluate.py
or paired_orchestrate_v2.py

Because JSONL result rows do not themselves contain model name, this script infers the model from the filename by default.

Example filenames:

gpt_4o_mini__chunking_v2__paired_results.jsonl
route_llm__chunking_v2__paired_results.jsonl

If you want stricter naming, keep that pattern.

Python script
import os
import re
import json
import math
import random
import argparse
from collections import defaultdict
from itertools import combinations


# ============================================================
# IO
# ============================================================

def load_jsonl(path):
    rows = []
    with open(path, "r", encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if line:
                rows.append(json.loads(line))
    return rows

def save_json(path, obj):
    with open(path, "w", encoding="utf-8") as f:
        json.dump(obj, f, indent=2, ensure_ascii=False)

def save_text(path, text):
    with open(path, "w", encoding="utf-8") as f:
        f.write(text)


# ============================================================
# Helpers
# ============================================================

def mean(xs):
    xs = [x for x in xs if x is not None]
    return sum(xs) / len(xs) if xs else None

def percentile(xs, p):
    if not xs:
        return None
    xs = sorted(xs)
    k = (len(xs) - 1) * p
    f = math.floor(k)
    c = math.ceil(k)
    if f == c:
        return xs[int(k)]
    return xs[f] * (c - k) + xs[c] * (k - f)

def bootstrap_ci(values, n_boot=5000, alpha=0.05, seed=42):
    vals = [v for v in values if v is not None]
    if not vals:
        return {"estimate": None, "ci_low": None, "ci_high": None, "n": 0}
    rng = random.Random(seed)
    n = len(vals)
    boots = []
    for _ in range(n_boot):
        sample = [vals[rng.randrange(n)] for _ in range(n)]
        boots.append(mean(sample))
    return {
        "estimate": mean(vals),
        "ci_low": percentile(boots, alpha / 2),
        "ci_high": percentile(boots, 1 - alpha / 2),
        "n": n,
    }

def fmt(x, digits=4):
    if x is None:
        return "n/a"
    return f"{x:.{digits}f}"

def infer_model_name(path):
    base = os.path.basename(path)
    if "__" in base:
        return base.split("__")[0]
    return os.path.splitext(base)[0]

def infer_experiment(rows):
    experiments = {r.get("experiment") for r in rows}
    if len(experiments) != 1:
        raise ValueError(f"Expected one experiment per file, got: {experiments}")
    return next(iter(experiments))

def group_by_pair(rows):
    grouped = defaultdict(list)
    for r in rows:
        grouped[r["pair_id"]].append(r)
    return grouped


# ============================================================
# Per-pair effect extraction
# ============================================================

def pair_effects_chunking(rows):
    grouped = group_by_pair(rows)
    effects = {}
    for pair_id, pair_rows in grouped.items():
        by_variant = {r["variant"]: r for r in pair_rows}
        if "flat" in by_variant and "chunked" in by_variant:
            effects[pair_id] = by_variant["chunked"]["correct"] - by_variant["flat"]["correct"]
    return effects

def pair_effects_interactivity(rows):
    grouped = group_by_pair(rows)
    effects = {}
    for pair_id, pair_rows in grouped.items():
        by_variant = {r["variant"]: r for r in pair_rows}
        if "low" in by_variant and "high" in by_variant:
            effects[pair_id] = by_variant["low"]["correct"] - by_variant["high"]["correct"]
    return effects

def pair_effects_extraneous(rows):
    grouped = group_by_pair(rows)
    effects = {
        "irrelevant_minus_confusable": {},
        "irrelevant_minus_contradictory_adjacent": {},
        "confusable_minus_contradictory_adjacent": {},
    }

    for pair_id, pair_rows in grouped.items():
        by_variant = {r["variant"]: r for r in pair_rows}
        need = ["irrelevant", "confusable", "contradictory_adjacent"]
        if not all(v in by_variant for v in need):
            continue

        ir = by_variant["irrelevant"]["correct"]
        co = by_variant["confusable"]["correct"]
        ca = by_variant["contradictory_adjacent"]["correct"]

        effects["irrelevant_minus_confusable"][pair_id] = ir - co
        effects["irrelevant_minus_contradictory_adjacent"][pair_id] = ir - ca
        effects["confusable_minus_contradictory_adjacent"][pair_id] = co - ca

    return effects


# ============================================================
# Cross-model comparison
# ============================================================

def compare_effect_maps(effect_map_a, effect_map_b):
    shared = sorted(set(effect_map_a.keys()) & set(effect_map_b.keys()))
    diffs = [effect_map_a[k] - effect_map_b[k] for k in shared]
    ci = bootstrap_ci(diffs)
    return {
        "n_shared_pairs": len(shared),
        "model_a_mean_effect": mean([effect_map_a[k] for k in shared]),
        "model_b_mean_effect": mean([effect_map_b[k] for k in shared]),
        "difference_a_minus_b": mean(diffs),
        "difference_ci": ci,
    }

def compare_models_for_experiment(model_rows):
    """
    model_rows: dict model_name -> rows
    """
    models = sorted(model_rows.keys())
    experiment = infer_experiment(next(iter(model_rows.values())))

    result = {
        "experiment": experiment,
        "models": models,
        "pairwise_comparisons": [],
    }

    if experiment == "chunking_v2":
        effect_maps = {m: pair_effects_chunking(rows) for m, rows in model_rows.items()}
        metric_name = "chunking_gain"

        for a, b in combinations(models, 2):
            comp = compare_effect_maps(effect_maps[a], effect_maps[b])
            comp["metric"] = metric_name
            comp["model_a"] = a
            comp["model_b"] = b
            result["pairwise_comparisons"].append(comp)

    elif experiment == "element_interactivity_v2":
        effect_maps = {m: pair_effects_interactivity(rows) for m, rows in model_rows.items()}
        metric_name = "interactivity_penalty"

        for a, b in combinations(models, 2):
            comp = compare_effect_maps(effect_maps[a], effect_maps[b])
            comp["metric"] = metric_name
            comp["model_a"] = a
            comp["model_b"] = b
            result["pairwise_comparisons"].append(comp)

    elif experiment == "extraneous_load_v2":
        effect_maps_by_metric = {}
        for m, rows in model_rows.items():
            effect_maps_by_metric[m] = pair_effects_extraneous(rows)

        metrics = [
            "irrelevant_minus_confusable",
            "irrelevant_minus_contradictory_adjacent",
            "confusable_minus_contradictory_adjacent",
        ]

        for metric in metrics:
            for a, b in combinations(models, 2):
                comp = compare_effect_maps(effect_maps_by_metric[a][metric], effect_maps_by_metric[b][metric])
                comp["metric"] = metric
                comp["model_a"] = a
                comp["model_b"] = b
                result["pairwise_comparisons"].append(comp)

    else:
        raise ValueError(f"Unsupported experiment: {experiment}")

    return result


# ============================================================
# Organizing inputs
# ============================================================

def load_grouped_by_experiment(paths):
    grouped = defaultdict(dict)

    for path in paths:
        rows = load_jsonl(path)
        experiment = infer_experiment(rows)
        model = infer_model_name(path)
        grouped[experiment][model] = rows

    return grouped


# ============================================================
# Markdown formatter
# ============================================================

def to_markdown(all_results):
    lines = ["# Cross-Model Significance Summary", ""]

    for experiment, result in all_results.items():
        lines.append(f"## {experiment}")
        lines.append("")

        headers = [
            "Metric",
            "Model A",
            "Model B",
            "Shared Pairs",
            "A Effect",
            "B Effect",
            "A - B",
            "95% CI Low",
            "95% CI High",
        ]
        lines.append("| " + " | ".join(headers) + " |")
        lines.append("|" + "|".join(["---"] * len(headers)) + "|")

        for row in result["pairwise_comparisons"]:
            ci = row["difference_ci"]
            lines.append(
                "| " + " | ".join([
                    str(row["metric"]),
                    str(row["model_a"]),
                    str(row["model_b"]),
                    str(row["n_shared_pairs"]),
                    fmt(row["model_a_mean_effect"]),
                    fmt(row["model_b_mean_effect"]),
                    fmt(row["difference_a_minus_b"]),
                    fmt(ci["ci_low"]),
                    fmt(ci["ci_high"]),
                ]) + " |"
            )

        lines.append("")

    return "\n".join(lines)


# ============================================================
# Main
# ============================================================

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--inputs",
        nargs="+",
        required=True,
        help="Detailed paired result JSONL files from multiple models"
    )
    parser.add_argument("--json-out", default="cross_model_significance.json")
    parser.add_argument("--md-out", default="cross_model_significance.md")
    args = parser.parse_args()

    grouped = load_grouped_by_experiment(args.inputs)

    all_results = {}
    for experiment, model_rows in grouped.items():
        if len(model_rows) < 2:
            print(f"Skipping {experiment}: need at least 2 models")
            continue
        all_results[experiment] = compare_models_for_experiment(model_rows)

    save_json(args.json_out, all_results)
    save_text(args.md_out, to_markdown(all_results))

    print(f"Saved JSON to {args.json_out}")
    print(f"Saved markdown to {args.md_out}")


if __name__ == "__main__":
    main()

Example usage
python cross_model_significance.py \
  --inputs \
    paired_v2_runs/gpt_4o_mini__chunking_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4_1_mini__chunking_v2__paired_results.jsonl \
    paired_v2_runs/route_llm__chunking_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4o_mini__element_interactivity_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4_1_mini__element_interactivity_v2__paired_results.jsonl \
    paired_v2_runs/route_llm__element_interactivity_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4o_mini__extraneous_load_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4_1_mini__extraneous_load_v2__paired_results.jsonl \
    paired_v2_runs/route_llm__extraneous_load_v2__paired_results.jsonl \
  --json-out cross_model_significance.json \
  --md-out cross_model_significance.md

What the outputs mean

Suppose you get:

Model A chunking gain: 0.18
Model B chunking gain: 0.07
Difference A - B = 0.11
95% CI: [0.05, 0.17]

Interpretation:

Model A benefits significantly more from chunking than Model B
that suggests A is more structure-sensitive, or B is more robust to flat presentation

If interactivity penalty is larger for a smaller model, that supports your bounded-capacity thesis.

If extraneous-load gaps are larger for a given model, that model is more vulnerable to misleading interference.

Important limitation

This is a bootstrap CI on paired effect differences, not a formal null-hypothesis exact test.

That is often enough for strong empirical analysis, but if you want publication-grade testing, next upgrades would be:

permutation tests on paired model-effect differences
exact or asymptotic tests for correlated proportions
multiple-comparison correction
Best next upgrade

The strongest next artifact would be either:

permutation test version

strongest statistical rigor for model-vs-model comparisons

publication figure compositor

assemble all benchmark plots into one paper figure

paper-ready results memo

auto-generate a concise narrative from the output tables

If you want the statistically strongest next move, ask for:

“Write the permutation test version.”
