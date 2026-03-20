Multiple comparison correction

Yes. This is exactly the next thing you want if you're moving from “interesting empirical signal” to “defensible statistical analysis.”

When you compare:

many model pairs
across several benchmarks
across several metrics

you create a multiple testing problem.
Without correction, you’ll overstate significance.

What this version adds

This upgraded script keeps the permutation-test core and adds:

Benjamini–Hochberg FDR correction
Bonferroni correction
optional Holm–Bonferroni correction

So you get, for every cross-model comparison:

raw permutation p-value
BH-adjusted p-value
Bonferroni-adjusted p-value
Holm-adjusted p-value
significance flags at your chosen alpha
Why these three
Benjamini–Hochberg

Best when you want to control false discovery rate and keep reasonable power.

Use this for:

exploratory or benchmark-heavy model comparison
Bonferroni

Very conservative. Controls family-wise error rate.

Use this when:

you want a hard-to-argue-with claim
number of tests is modest
Holm

Usually better than plain Bonferroni while still controlling family-wise error rate.

Use this when:

you want a strong confirmatory correction without being quite as brutal as Bonferroni
Important design choice

This script applies correction across all pairwise tests in the run by default.

That means if you test:

3 chunking model comparisons
3 interactivity comparisons
9 extraneous comparisons

all of those p-values are corrected together.

That is the safest default.

If later you want, we can add:

correction within experiment only
correction within metric family only
Python script
import os
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
    diffs = [d for d in diffs if d is not None]
    if not diffs:
        return {
            "observed_mean": None,
            "p_value_two_sided": None,
            "n": 0,
            "n_perm": n_perm,
        }

    rng = random.Random(seed)
    observed = mean(diffs)
    extreme = 0

    for _ in range(n_perm):
        stat = mean([d if rng.random() < 0.5 else -d for d in diffs])
        if abs(stat) >= abs(observed):
            extreme += 1

    p = (extreme + 1) / (n_perm + 1)

    return {
        "observed_mean": observed,
        "p_value_two_sided": p,
        "n": len(diffs),
        "n_perm": n_perm,
    }


# ============================================================
# Multiple comparison correction
# ============================================================

def bonferroni_adjust(pvals):
    m = len(pvals)
    return [min(1.0, p * m) if p is not None else None for p in pvals]

def holm_adjust(pvals):
    indexed = [(i, p) for i, p in enumerate(pvals) if p is not None]
    m = len(indexed)
    out = [None] * len(pvals)
    if m == 0:
        return out

    indexed.sort(key=lambda x: x[1])

    adjusted = [0.0] * m
    for rank, (orig_idx, p) in enumerate(indexed, start=1):
        adjusted[rank - 1] = min(1.0, (m - rank + 1) * p)

    # enforce monotonicity
    for i in range(1, m):
        adjusted[i] = max(adjusted[i], adjusted[i - 1])

    for (ranked_item, adj_p) in zip(indexed, adjusted):
        orig_idx = ranked_item[0]
        out[orig_idx] = min(1.0, adj_p)

    return out

def bh_adjust(pvals):
    indexed = [(i, p) for i, p in enumerate(pvals) if p is not None]
    m = len(indexed)
    out = [None] * len(pvals)
    if m == 0:
        return out

    indexed.sort(key=lambda x: x[1])
    adjusted = [0.0] * m

    for rank, (_, p) in enumerate(indexed, start=1):
        adjusted[rank - 1] = p * m / rank

    # enforce monotonicity from right to left
    for i in range(m - 2, -1, -1):
        adjusted[i] = min(adjusted[i], adjusted[i + 1])

    for (orig, _), adj in zip(indexed, adjusted):
        out[orig] = min(1.0, adj)

    return out


# ============================================================
# Core comparison
# ============================================================

def compare_effect_maps(effect_map_a, effect_map_b, n_perm=20000, seed=42):
    shared = sorted(set(effect_map_a.keys()) & set(effect_map_b.keys()))
    a_vals = [effect_map_a[k] for k in shared]
    b_vals = [effect_map_b[k] for k in shared]
    diffs = [a - b for a, b in zip(a_vals, b_vals)]

    ci = bootstrap_ci(diffs, seed=seed)
    perm = sign_flip_permutation_test(diffs, n_perm=n_perm, seed=seed)

    return {
        "n_shared_pairs": len(shared),
        "model_a_mean_effect": mean(a_vals),
        "model_b_mean_effect": mean(b_vals),
        "difference_a_minus_b": mean(diffs),
        "difference_ci": ci,
        "permutation_test": perm,
    }

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
        for a, b in combinations(models, 2):
            comp = compare_effect_maps(effect_maps[a], effect_maps[b], n_perm=n_perm, seed=seed)
            comp["metric"] = "chunking_gain"
            comp["model_a"] = a
            comp["model_b"] = b
            comp["experiment"] = experiment
            result["pairwise_comparisons"].append(comp)

    elif experiment == "element_interactivity_v2":
        effect_maps = {m: pair_effects_interactivity(rows) for m, rows in model_rows.items()}
        for a, b in combinations(models, 2):
            comp = compare_effect_maps(effect_maps[a], effect_maps[b], n_perm=n_perm, seed=seed)
            comp["metric"] = "interactivity_penalty"
            comp["model_a"] = a
            comp["model_b"] = b
            comp["experiment"] = experiment
            result["pairwise_comparisons"].append(comp)

    elif experiment == "extraneous_load_v2":
        effect_maps = {m: pair_effects_extraneous(rows) for m, rows in model_rows.items()}
        metrics = [
            "irrelevant_minus_confusable",
            "irrelevant_minus_contradictory_adjacent",
            "confusable_minus_contradictory_adjacent",
        ]
        for metric in metrics:
            for a, b in combinations(models, 2):
                comp = compare_effect_maps(effect_maps[a][metric], effect_maps[b][metric], n_perm=n_perm, seed=seed)
                comp["metric"] = metric
                comp["model_a"] = a
                comp["model_b"] = b
                comp["experiment"] = experiment
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
# Correction application
# ============================================================

def flatten_comparisons(all_results):
    flat = []
    for experiment, result in all_results.items():
        for row in result["pairwise_comparisons"]:
            flat.append(row)
    return flat

def apply_corrections(all_results, alpha=0.05):
    flat = flatten_comparisons(all_results)
    raw_pvals = [
        row.get("permutation_test", {}).get("p_value_two_sided")
        for row in flat
    ]

    bonf = bonferroni_adjust(raw_pvals)
    holm = holm_adjust(raw_pvals)
    bh = bh_adjust(raw_pvals)

    for row, p_bonf, p_holm, p_bh in zip(flat, bonf, holm, bh):
        row["multiple_comparison_correction"] = {
            "alpha": alpha,
            "raw_p": row.get("permutation_test", {}).get("p_value_two_sided"),
            "bonferroni_p": p_bonf,
            "holm_p": p_holm,
            "bh_fdr_p": p_bh,
            "significant_raw": (row.get("permutation_test", {}).get("p_value_two_sided") is not None and row["permutation_test"]["p_value_two_sided"] < alpha),
            "significant_bonferroni": (p_bonf is not None and p_bonf < alpha),
            "significant_holm": (p_holm is not None and p_holm < alpha),
            "significant_bh_fdr": (p_bh is not None and p_bh < alpha),
        }

    return all_results


# ============================================================
# Markdown
# ============================================================

def to_markdown(all_results):
    lines = ["# Cross-Model Permutation Test Summary with Multiple Comparison Correction", ""]

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
            "A-B",
            "CI Low",
            "CI High",
            "Raw p",
            "BH p",
            "Holm p",
            "Bonf p",
            "BH sig",
            "Holm sig",
            "Bonf sig",
        ]
        lines.append("| " + " | ".join(headers) + " |")
        lines.append("|" + "|".join(["---"] * len(headers)) + "|")

        for row in result["pairwise_comparisons"]:
            ci = row["difference_ci"]
            perm = row["permutation_test"]
            corr = row["multiple_comparison_correction"]

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
                    fmt(corr["raw_p"]),
                    fmt(corr["bh_fdr_p"]),
                    fmt(corr["holm_p"]),
                    fmt(corr["bonferroni_p"]),
                    str(corr["significant_bh_fdr"]),
                    str(corr["significant_holm"]),
                    str(corr["significant_bonferroni"]),
                ]) + " |"
            )

        lines.append("")

    return "\n".join(lines)


# ============================================================
# Main
# ============================================================

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--inputs", nargs="+", required=True, help="Detailed paired result JSONL files from multiple models")
    parser.add_argument("--json-out", default="cross_model_permutation_corrected.json")
    parser.add_argument("--md-out", default="cross_model_permutation_corrected.md")
    parser.add_argument("--n-perm", type=int, default=20000)
    parser.add_argument("--seed", type=int, default=42)
    parser.add_argument("--alpha", type=float, default=0.05)
    args = parser.parse_args()

    grouped = load_grouped_by_experiment(args.inputs)

    all_results = {}
    for experiment, model_rows in grouped.items():
        if len(model_rows) < 2:
            print(f"Skipping {experiment}: need at least 2 models")
            continue
        all_results[experiment] = compare_models_for_experiment(
            model_rows=model_rows,
            n_perm=args.n_perm,
            seed=args.seed,
        )

    all_results = apply_corrections(all_results, alpha=args.alpha)

    save_json(args.json_out, all_results)
    save_text(args.md_out, to_markdown(all_results))

    print(f"Saved JSON to {args.json_out}")
    print(f"Saved markdown to {args.md_out}")


if __name__ == "__main__":
    main()

Example usage
python cross_model_permutation_corrected.py \
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
  --json-out cross_model_permutation_corrected.json \
  --md-out cross_model_permutation_corrected.md \
  --n-perm 20000 \
  --alpha 0.05

How to interpret corrected results
Raw p

The uncorrected permutation p-value.

BH p

The false-discovery-rate adjusted p-value.

Best for:

keeping power
multi-benchmark exploratory analysis
Holm p

Family-wise error rate control, less conservative than Bonferroni.

Best for:

stronger confirmatory claims
Bonf p

Very conservative family-wise error rate control.

Best for:

strongest skepticism standard
Recommended reporting language

If a result survives BH but not Holm/Bonferroni:

The effect remained significant under FDR correction but not under family-wise error rate controls, suggesting a likely but not maximally conservative effect.

If a result survives Holm:

The effect remained significant under Holm correction, supporting a robust cross-model difference after multiple comparison control.

If a result survives Bonferroni:

The effect remained significant under Bonferroni correction, indicating a highly robust cross-model difference.

Best next upgrade

Now that the stats layer is strong, the best next artifact is probably one of:

exact p-value option for small n
within-family correction modes
paper-style results memo generator
forest plot script for corrected comparisons

My recommendation:

forest plot script for corrected comparisons

because it turns all this statistics work into something instantly legible.

If you want, I can write that next.
