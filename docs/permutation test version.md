Permutation test version

Yes. This is the right rigor upgrade.

A permutation test is stronger here because it directly tests whether the cross-model paired effect difference could plausibly arise by chance under label exchangeability.

For your setup, this is especially appropriate because:

both models are evaluated on the same paired items
each paired item yields a per-pair effect
the null is that model A and model B are exchangeable on those per-pair effects

That gives you a clean nonparametric comparison.

What this script does

For each experiment and each model pair, it computes:

the observed mean difference in paired effects:
ΔA−B​=E[ei(A)​−ei(B)​]
a two-sided permutation p-value
a sign-flip permutation test
bootstrap CI for the same difference
shared pair count
Why sign-flip is appropriate here

For each shared pair, define:

di​=ei(A)​−ei(B)​

Under the null that the two models are exchangeable on that paired item, the sign of di​ is arbitrary.

So the permutation procedure is:

randomly flip the sign of each di​
recompute the mean
compare the observed absolute mean to the null distribution

This is efficient and statistically natural for matched-pair differences.

Supported metrics
chunking_v2
chunking_gain
element_interactivity_v2
interactivity_penalty
extraneous_load_v2
irrelevant_minus_confusable
irrelevant_minus_contradictory_adjacent
confusable_minus_contradictory_adjacent
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

def fmt(x, digits=6):
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
# Effect extraction
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
# Permutation test
# ============================================================

def sign_flip_permutation_test(diffs, n_perm=20000, seed=42):
    """
    Two-sided sign-flip permutation test on paired differences.
    """
    diffs = [d for d in diffs if d is not None]
    if not diffs:
        return {
            "observed_mean": None,
            "p_value_two_sided": None,
            "n": 0,
        }

    rng = random.Random(seed)
    observed = mean(diffs)
    n = len(diffs)

    extreme = 0
    for _ in range(n_perm):
        flipped = [d if rng.random() < 0.5 else -d for d in diffs]
        stat = mean(flipped)
        if abs(stat) >= abs(observed):
            extreme += 1

    # add-one correction
    p = (extreme + 1) / (n_perm + 1)

    return {
        "observed_mean": observed,
        "p_value_two_sided": p,
        "n": n,
        "n_perm": n_perm,
    }


# ============================================================
# Cross-model comparison core
# ============================================================

def compare_effect_maps(effect_map_a, effect_map_b, n_perm=20000, seed=42):
    shared = sorted(set(effect_map_a.keys()) & set(effect_map_b.keys()))
    model_a_effects = [effect_map_a[k] for k in shared]
    model_b_effects = [effect_map_b[k] for k in shared]
    diffs = [a - b for a, b in zip(model_a_effects, model_b_effects)]

    ci = bootstrap_ci(diffs, seed=seed)
    perm = sign_flip_permutation_test(diffs, n_perm=n_perm, seed=seed)

    return {
        "n_shared_pairs": len(shared),
        "model_a_mean_effect": mean(model_a_effects),
        "model_b_mean_effect": mean(model_b_effects),
        "difference_a_minus_b": mean(diffs),
        "difference_ci": ci,
        "permutation_test": perm,
    }


# ============================================================
# Experiment-level comparison
# ============================================================

def compare_models_for_experiment(model_rows, n_perm=20000, seed=42):
    models = sorted(model_rows.keys())
    experiment = infer_experiment(next(iter(model_rows.values())))

    result = {
        "experiment": experiment,
        "models": models,
        "pairwise_comparisons": [],
    }

    if experiment == "chunking_v2":
        effect_maps = {m: pair_effects_chunking(rows) for m, rows in model_rows.items()}
        metric_names = ["chunking_gain"]

        for a, b in combinations(models, 2):
            comp = compare_effect_maps(effect_maps[a], effect_maps[b], n_perm=n_perm, seed=seed)
            comp["metric"] = "chunking_gain"
            comp["model_a"] = a
            comp["model_b"] = b
            result["pairwise_comparisons"].append(comp)

    elif experiment == "element_interactivity_v2":
        effect_maps = {m: pair_effects_interactivity(rows) for m, rows in model_rows.items()}

        for a, b in combinations(models, 2):
            comp = compare_effect_maps(effect_maps[a], effect_maps[b], n_perm=n_perm, seed=seed)
            comp["metric"] = "interactivity_penalty"
            comp["model_a"] = a
            comp["model_b"] = b
            result["pairwise_comparisons"].append(comp)

    elif experiment == "extraneous_load_v2":
        effect_maps = {m: pair_effects_extraneous(rows) for m, rows in model_rows.items()}
        metric_names = [
            "irrelevant_minus_confusable",
            "irrelevant_minus_contradictory_adjacent",
            "confusable_minus_contradictory_adjacent",
        ]

        for metric in metric_names:
            for a, b in combinations(models, 2):
                comp = compare_effect_maps(effect_maps[a][metric], effect_maps[b][metric], n_perm=n_perm, seed=seed)
                comp["metric"] = metric
                comp["model_a"] = a
                comp["model_b"] = b
                result["pairwise_comparisons"].append(comp)

    else:
        raise ValueError(f"Unsupported experiment: {experiment}")

    return result


# ============================================================
# Input organization
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
    lines = ["# Cross-Model Permutation Test Summary", ""]

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
            "CI Low",
            "CI High",
            "Permutation p",
        ]
        lines.append("| " + " | ".join(headers) + " |")
        lines.append("|" + "|".join(["---"] * len(headers)) + "|")

        for row in result["pairwise_comparisons"]:
            ci = row["difference_ci"]
            perm = row["permutation_test"]

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
                    fmt(perm["p_value_two_sided"]),
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
    parser.add_argument("--json-out", default="cross_model_permutation.json")
    parser.add_argument("--md-out", default="cross_model_permutation.md")
    parser.add_argument("--n-perm", type=int, default=20000)
    parser.add_argument("--seed", type=int, default=42)
    args = parser.parse_args()

    grouped = load_grouped_by_experiment(args.inputs)

    all_results = {}
    for experiment, model_rows in grouped.items():
        if len(model_rows) < 2:
            print(f"Skipping {experiment}: need at least 2 models")
            continue
        all_results[experiment] = compare_models_for_experiment(
            model_rows,
            n_perm=args.n_perm,
            seed=args.seed,
        )

    save_json(args.json_out, all_results)
    save_text(args.md_out, to_markdown(all_results))

    print(f"Saved JSON to {args.json_out}")
    print(f"Saved markdown to {args.md_out}")


if __name__ == "__main__":
    main()

Example usage
python cross_model_permutation.py \
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
  --json-out cross_model_permutation.json \
  --md-out cross_model_permutation.md \
  --n-perm 20000 \
  --seed 42

How to interpret it

Suppose for chunking you get:

Model A effect: 0.180000
Model B effect: 0.060000
A - B: 0.120000
CI: [0.040000, 0.190000]
permutation p: 0.003200

That means:

Model A benefits more from chunking than Model B
the difference is unlikely under the null of itemwise exchangeability
that is strong evidence of a real cross-model difference in structure sensitivity
Why this is better than just bootstrap alone

Bootstrap gives you:

uncertainty interval around the effect difference

Permutation gives you:

a direct null-based significance test

Together, they are much stronger.

Remaining statistical caveats

This version still does not include:

multiple comparison correction
exact enumeration for very small n
hierarchical modeling across experiments
calibration for dependence across multiple metrics from the same benchmark family

Those are advanced upgrades.

Best next upgrade

The best next move depends on your goal:

If your goal is scientific rigor

Ask for: “Add multiple comparison correction and exact p-value options.”

If your goal is publication

Ask for: “Write a paper-style results memo from the permutation outputs.”

If your goal is workflow speed

Ask for: “Write a Makefile / one-command pipeline for the full benchmark.”

My recommendation:
multiple comparison correction is the strongest next step.
