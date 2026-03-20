Paired evaluator and paired stats pipeline

Below is a practical two-part pipeline for your v2 paired benchmarks.

It gives you:

paired evaluator

reads gold JSONL + predictions JSONL
grades each example
preserves pair_id, variant, correctness, and metadata
writes a detailed per-example results file

paired stats analyzer

reads the detailed paired results
computes paired deltas by pair_id
reports:
chunking gain
interactivity penalty
extraneous distractor sensitivity
includes:
paired bootstrap confidence intervals
McNemar-style matched outcome counts
markdown summary output

This is the scientifically meaningful version for the v2 datasets.

Expected inputs
Gold JSONL

From your v2 generator, each row should include fields like:

id
pair_id
variant
experiment
prompt
target
condition
grader
Predictions JSONL

Format:

{"id": "chunk_v2_pair_000001_flat", "prediction": "blue"}
{"id": "chunk_v2_pair_000001_chunked", "prediction": "blue"}

Part 1: Paired evaluator
import json
import re
import argparse
from typing import Dict, Any, List


# ============================================================
# IO
# ============================================================

def load_jsonl(path: str) -> List[Dict[str, Any]]:
    rows = []
    with open(path, "r", encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if line:
                rows.append(json.loads(line))
    return rows

def write_jsonl(path: str, rows: List[Dict[str, Any]]) -> None:
    with open(path, "w", encoding="utf-8") as f:
        for row in rows:
            f.write(json.dumps(row, ensure_ascii=False) + "\n")


# ============================================================
# Normalization
# ============================================================

def normalize_text(s: str) -> str:
    if s is None:
        return ""
    s = str(s).strip()
    s = re.sub(r"\s+", " ", s)
    return s

def normalize_exact(s: str) -> str:
    return normalize_text(s)


# ============================================================
# Grading
# ============================================================

def grade_exact(prediction: str, accepted_answers: List[str]) -> bool:
    pred = normalize_exact(prediction)
    accepted = {normalize_exact(a) for a in accepted_answers}
    return pred in accepted

def evaluate_example(gold: Dict[str, Any], pred_map: Dict[str, str]) -> Dict[str, Any]:
    pred = pred_map.get(gold["id"], "")
    grader = gold["grader"]

    if grader["type"] == "exact_match_normalized":
        accepted = grader.get("accepted_answers", [gold["target"]])
        correct = grade_exact(pred, accepted)
    else:
        raise ValueError(f"Unsupported grader type for paired v2 pipeline: {grader['type']}")

    return {
        "id": gold["id"],
        "pair_id": gold.get("pair_id"),
        "base_item_id": gold.get("base_item_id"),
        "experiment": gold.get("experiment"),
        "variant": gold.get("variant"),
        "condition": gold.get("condition", {}),
        "target": gold.get("target"),
        "prediction": pred,
        "correct": int(correct),
        "missing_prediction": int(gold["id"] not in pred_map),
    }


# ============================================================
# Main
# ============================================================

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--gold", required=True, help="Gold JSONL")
    parser.add_argument("--pred", required=True, help="Predictions JSONL")
    parser.add_argument("--out", required=True, help="Detailed paired results JSONL")
    args = parser.parse_args()

    gold_rows = load_jsonl(args.gold)
    pred_rows = load_jsonl(args.pred)

    pred_map = {row["id"]: row.get("prediction", "") for row in pred_rows if "id" in row}

    detailed = [evaluate_example(g, pred_map) for g in gold_rows]
    write_jsonl(args.out, detailed)

    total = len(detailed)
    acc = sum(r["correct"] for r in detailed) / total if total else 0.0
    missing = sum(r["missing_prediction"] for r in detailed)

    print(f"Saved detailed paired results to {args.out}")
    print(f"Examples: {total}")
    print(f"Accuracy: {acc:.4f}")
    print(f"Missing predictions: {missing}")

if __name__ == "__main__":
    main()

Example usage
python paired_evaluate.py \
  --gold chunking_v2_test.jsonl \
  --pred chunk_preds.jsonl \
  --out chunk_paired_results.jsonl

Output format

Each result row looks like:

{
  "id": "chunk_v2_pair_000001_flat",
  "pair_id": "chunk_v2_pair_000001",
  "base_item_id": 1,
  "experiment": "chunking_v2",
  "variant": "flat",
  "condition": {"format": "flat", "query_type": "lookup", "num_facts": 5},
  "target": "blue",
  "prediction": "blue",
  "correct": 1,
  "missing_prediction": 0
}

Part 2: Paired stats analyzer
import json
import math
import random
import argparse
from collections import defaultdict


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

def group_by_pair(rows):
    grouped = defaultdict(list)
    for row in rows:
        grouped[row["pair_id"]].append(row)
    return grouped


# ============================================================
# McNemar helpers
# ============================================================

def mcnemar_counts(pairs, variant_a, variant_b):
    b = 0  # a correct, b wrong
    c = 0  # a wrong, b correct

    for pair in pairs.values():
        by_variant = {r["variant"]: r for r in pair}
        if variant_a not in by_variant or variant_b not in by_variant:
            continue
        a_ok = by_variant[variant_a]["correct"]
        b_ok = by_variant[variant_b]["correct"]

        if a_ok == 1 and b_ok == 0:
            b += 1
        elif a_ok == 0 and b_ok == 1:
            c += 1

    return {"a_correct_b_wrong": b, "a_wrong_b_correct": c}

def mcnemar_chi_square(counts):
    b = counts["a_correct_b_wrong"]
    c = counts["a_wrong_b_correct"]
    if b + c == 0:
        return None
    return ((abs(b - c) - 1) ** 2) / (b + c)


# ============================================================
# Chunking analysis
# ============================================================

def analyze_chunking(rows):
    pairs = group_by_pair(rows)
    deltas = []
    flat_acc = []
    chunked_acc = []

    for pair_id, pair_rows in pairs.items():
        by_variant = {r["variant"]: r for r in pair_rows}
        if "flat" not in by_variant or "chunked" not in by_variant:
            continue

        flat = by_variant["flat"]["correct"]
        chunked = by_variant["chunked"]["correct"]

        flat_acc.append(flat)
        chunked_acc.append(chunked)
        deltas.append(chunked - flat)

    ci = bootstrap_ci(deltas)
    counts = mcnemar_counts(pairs, "chunked", "flat")
    chi2 = mcnemar_chi_square(counts)

    return {
        "experiment": "chunking_v2",
        "n_pairs": len(deltas),
        "flat_accuracy": mean(flat_acc),
        "chunked_accuracy": mean(chunked_acc),
        "paired_gain": mean(deltas),
        "paired_gain_ci": ci,
        "mcnemar_counts": counts,
        "mcnemar_chi_square_cc": chi2,
    }


# ============================================================
# Interactivity analysis
# ============================================================

def analyze_interactivity(rows):
    pairs = group_by_pair(rows)
    deltas = []
    low_acc = []
    high_acc = []

    for pair_id, pair_rows in pairs.items():
        by_variant = {r["variant"]: r for r in pair_rows}
        if "low" not in by_variant or "high" not in by_variant:
            continue

        low = by_variant["low"]["correct"]
        high = by_variant["high"]["correct"]

        low_acc.append(low)
        high_acc.append(high)
        deltas.append(low - high)

    ci = bootstrap_ci(deltas)
    counts = mcnemar_counts(pairs, "low", "high")
    chi2 = mcnemar_chi_square(counts)

    return {
        "experiment": "element_interactivity_v2",
        "n_pairs": len(deltas),
        "low_accuracy": mean(low_acc),
        "high_accuracy": mean(high_acc),
        "paired_penalty": mean(deltas),
        "paired_penalty_ci": ci,
        "mcnemar_counts": counts,
        "mcnemar_chi_square_cc": chi2,
    }


# ============================================================
# Extraneous analysis
# ============================================================

def analyze_extraneous(rows):
    pairs = group_by_pair(rows)

    irrelevant_acc = []
    confusable_acc = []
    contradictory_acc = []

    delta_irrel_conf = []
    delta_irrel_contra = []
    delta_conf_contra = []

    valid_pairs = 0
    for pair_id, pair_rows in pairs.items():
        by_variant = {r["variant"]: r for r in pair_rows}
        needed = ["irrelevant", "confusable", "contradictory_adjacent"]
        if not all(v in by_variant for v in needed):
            continue

        ir = by_variant["irrelevant"]["correct"]
        co = by_variant["confusable"]["correct"]
        ca = by_variant["contradictory_adjacent"]["correct"]

        irrelevant_acc.append(ir)
        confusable_acc.append(co)
        contradictory_acc.append(ca)

        delta_irrel_conf.append(ir - co)
        delta_irrel_contra.append(ir - ca)
        delta_conf_contra.append(co - ca)
        valid_pairs += 1

    counts_irrel_conf = mcnemar_counts(pairs, "irrelevant", "confusable")
    counts_irrel_contra = mcnemar_counts(pairs, "irrelevant", "contradictory_adjacent")
    counts_conf_contra = mcnemar_counts(pairs, "confusable", "contradictory_adjacent")

    return {
        "experiment": "extraneous_load_v2",
        "n_pairs": valid_pairs,
        "irrelevant_accuracy": mean(irrelevant_acc),
        "confusable_accuracy": mean(confusable_acc),
        "contradictory_adjacent_accuracy": mean(contradictory_acc),
        "gap_irrelevant_minus_confusable": mean(delta_irrel_conf),
        "gap_irrelevant_minus_confusable_ci": bootstrap_ci(delta_irrel_conf),
        "gap_irrelevant_minus_contradictory_adjacent": mean(delta_irrel_contra),
        "gap_irrelevant_minus_contradictory_adjacent_ci": bootstrap_ci(delta_irrel_contra),
        "gap_confusable_minus_contradictory_adjacent": mean(delta_conf_contra),
        "gap_confusable_minus_contradictory_adjacent_ci": bootstrap_ci(delta_conf_contra),
        "mcnemar_irrelevant_vs_confusable": {
            "counts": counts_irrel_conf,
            "chi_square_cc": mcnemar_chi_square(counts_irrel_conf),
        },
        "mcnemar_irrelevant_vs_contradictory_adjacent": {
            "counts": counts_irrel_contra,
            "chi_square_cc": mcnemar_chi_square(counts_irrel_contra),
        },
        "mcnemar_confusable_vs_contradictory_adjacent": {
            "counts": counts_conf_contra,
            "chi_square_cc": mcnemar_chi_square(counts_conf_contra),
        },
    }


# ============================================================
# Dispatch
# ============================================================

def infer_experiment(rows):
    experiments = {r.get("experiment") for r in rows}
    if len(experiments) != 1:
        raise ValueError(f"Expected exactly one experiment in file, got: {experiments}")
    return next(iter(experiments))

def analyze(rows):
    experiment = infer_experiment(rows)

    if experiment == "chunking_v2":
        return analyze_chunking(rows)
    if experiment == "element_interactivity_v2":
        return analyze_interactivity(rows)
    if experiment == "extraneous_load_v2":
        return analyze_extraneous(rows)

    raise ValueError(f"Unsupported paired experiment: {experiment}")


# ============================================================
# Markdown output
# ============================================================

def to_markdown(result):
    exp = result["experiment"]
    lines = [f"# Paired Analysis: {exp}", ""]

    if exp == "chunking_v2":
        ci = result["paired_gain_ci"]
        lines += [
            f"- Pairs: `{result['n_pairs']}`",
            f"- Flat accuracy: `{fmt(result['flat_accuracy'])}`",
            f"- Chunked accuracy: `{fmt(result['chunked_accuracy'])}`",
            f"- Paired gain (`chunked - flat`): `{fmt(result['paired_gain'])}`",
            f"- Bootstrap CI: `[{fmt(ci['ci_low'])}, {fmt(ci['ci_high'])}]`",
            f"- McNemar counts: `{result['mcnemar_counts']}`",
            f"- McNemar chi-square (continuity corrected): `{fmt(result['mcnemar_chi_square_cc'])}`",
        ]

    elif exp == "element_interactivity_v2":
        ci = result["paired_penalty_ci"]
        lines += [
            f"- Pairs: `{result['n_pairs']}`",
            f"- Low-interactivity accuracy: `{fmt(result['low_accuracy'])}`",
            f"- High-interactivity accuracy: `{fmt(result['high_accuracy'])}`",
            f"- Paired penalty (`low - high`): `{fmt(result['paired_penalty'])}`",
            f"- Bootstrap CI: `[{fmt(ci['ci_low'])}, {fmt(ci['ci_high'])}]`",
            f"- McNemar counts: `{result['mcnemar_counts']}`",
            f"- McNemar chi-square (continuity corrected): `{fmt(result['mcnemar_chi_square_cc'])}`",
        ]

    elif exp == "extraneous_load_v2":
        ci1 = result["gap_irrelevant_minus_confusable_ci"]
        ci2 = result["gap_irrelevant_minus_contradictory_adjacent_ci"]
        ci3 = result["gap_confusable_minus_contradictory_adjacent_ci"]
        lines += [
            f"- Pairs: `{result['n_pairs']}`",
            f"- Irrelevant accuracy: `{fmt(result['irrelevant_accuracy'])}`",
            f"- Confusable accuracy: `{fmt(result['confusable_accuracy'])}`",
            f"- Contradictory-adjacent accuracy: `{fmt(result['contradictory_adjacent_accuracy'])}`",
            "",
            f"- Gap (`irrelevant - confusable`): `{fmt(result['gap_irrelevant_minus_confusable'])}`",
            f"- Bootstrap CI: `[{fmt(ci1['ci_low'])}, {fmt(ci1['ci_high'])}]`",
            f"- McNemar: `{result['mcnemar_irrelevant_vs_confusable']}`",
            "",
            f"- Gap (`irrelevant - contradictory_adjacent`): `{fmt(result['gap_irrelevant_minus_contradictory_adjacent'])}`",
            f"- Bootstrap CI: `[{fmt(ci2['ci_low'])}, {fmt(ci2['ci_high'])}]`",
            f"- McNemar: `{result['mcnemar_irrelevant_vs_contradictory_adjacent']}`",
            "",
            f"- Gap (`confusable - contradictory_adjacent`): `{fmt(result['gap_confusable_minus_contradictory_adjacent'])}`",
            f"- Bootstrap CI: `[{fmt(ci3['ci_low'])}, {fmt(ci3['ci_high'])}]`",
            f"- McNemar: `{result['mcnemar_confusable_vs_contradictory_adjacent']}`",
        ]

    lines.append("")
    return "\n".join(lines)


# ============================================================
# Main
# ============================================================

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", required=True, help="Detailed paired results JSONL")
    parser.add_argument("--json-out", default=None, help="Optional JSON output")
    parser.add_argument("--md-out", default=None, help="Optional markdown output")
    args = parser.parse_args()

    rows = load_jsonl(args.input)
    result = analyze(rows)

    if args.json_out:
        with open(args.json_out, "w", encoding="utf-8") as f:
            json.dump(result, f, indent=2, ensure_ascii=False)
        print(f"Saved JSON analysis to {args.json_out}")

    md = to_markdown(result)
    if args.md_out:
        with open(args.md_out, "w", encoding="utf-8") as f:
            f.write(md)
        print(f"Saved markdown analysis to {args.md_out}")
    else:
        print(md)

if __name__ == "__main__":
    main()

Example usage
Chunking v2
python paired_evaluate.py \
  --gold chunking_v2_test.jsonl \
  --pred chunk_preds.jsonl \
  --out chunk_paired_results.jsonl

python paired_stats.py \
  --input chunk_paired_results.jsonl \
  --json-out chunk_effects.json \
  --md-out chunk_effects.md

Extraneous load v2
python paired_evaluate.py \
  --gold extraneous_load_v2_test.jsonl \
  --pred extr_preds.jsonl \
  --out extr_paired_results.jsonl

python paired_stats.py \
  --input extr_paired_results.jsonl \
  --json-out extr_effects.json \
  --md-out extr_effects.md

Element interactivity v2
python paired_evaluate.py \
  --gold element_interactivity_v2_test.jsonl \
  --pred inter_preds.jsonl \
  --out inter_paired_results.jsonl

python paired_stats.py \
  --input inter_paired_results.jsonl \
  --json-out inter_effects.json \
  --md-out inter_effects.md

What the results mean
Chunking

If:

paired gain=E[chunked correct−flat correct]>0

then chunking improves performance on matched items.

Interactivity

If:

paired penalty=E[low correct−high correct]>0

then tightly coupled reasoning is harder than matched low-interactivity versions.

Extraneous load

If:

E[irrelevant−confusable]>0

and/or

E[irrelevant−contradictory adjacent]>0

then interference quality matters, not just extra text.

Notes on McNemar here

The script reports:

discordant matched counts
continuity-corrected chi-square statistic

It does not compute an exact p-value.

That’s okay for a research engineering workflow, but if you want publication-grade testing, the next upgrade is:

exact McNemar p-values
or permutation tests on paired deltas
Best next upgrade

The strongest next artifact would be a multi-model paired orchestrator that:

runs several models on v2 datasets
evaluates each
computes paired effects for each model
outputs a single comparison report like:
chunking gain by model
interactivity penalty by model
distractor sensitivity by model

If you want, I can write that next as a single script: paired_orchestrate_v2.py.
