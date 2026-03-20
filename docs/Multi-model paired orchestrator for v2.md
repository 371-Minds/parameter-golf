Multi-model paired orchestrator for v2

Excellent. This is the right consolidation step.

This script is designed to:

run multiple models
on multiple paired v2 benchmark files
through an OpenAI-compatible API
save predictions
evaluate paired correctness
compute paired effects
write a single comparison JSON
optionally write a markdown report

So this becomes your end-to-end empirical pipeline for the v2 benchmarks.

What it supports
Datasets
chunking_v2
extraneous_load_v2
element_interactivity_v2
Effects
chunking gain: chunked - flat
interactivity penalty: low - high
distractor sensitivity:
irrelevant - confusable
irrelevant - contradictory_adjacent
confusable - contradictory_adjacent
Outputs
per-model prediction JSONL files
per-model detailed paired result JSONL files
one combined paired_comparison.json
optional paired_report.md
Python script
import os
import re
import json
import math
import time
import random
import argparse
from collections import defaultdict

from openai import OpenAI


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

def write_jsonl(path, rows):
    with open(path, "w", encoding="utf-8") as f:
        for row in rows:
            f.write(json.dumps(row, ensure_ascii=False) + "\n")

def save_json(path, obj):
    with open(path, "w", encoding="utf-8") as f:
        json.dump(obj, f, indent=2, ensure_ascii=False)

def save_text(path, text):
    with open(path, "w", encoding="utf-8") as f:
        f.write(text)


# ============================================================
# Helpers
# ============================================================

def ensure_dir(path):
    os.makedirs(path, exist_ok=True)

def slugify(s):
    s = s.lower().strip()
    s = re.sub(r"[^a-z0-9]+", "_", s)
    s = re.sub(r"_+", "_", s).strip("_")
    return s

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

def normalize_text(s):
    if s is None:
        return ""
    s = str(s).strip()
    s = re.sub(r"\s+", " ", s)
    return s

def infer_experiment_name(dataset_path, rows):
    exps = {r.get("experiment") for r in rows}
    if len(exps) == 1 and next(iter(exps)) is not None:
        return next(iter(exps))
    name = os.path.basename(dataset_path).lower()
    if "chunk" in name:
        return "chunking_v2"
    if "extraneous" in name:
        return "extraneous_load_v2"
    if "interactivity" in name:
        return "element_interactivity_v2"
    return "unknown_experiment"

def group_by_pair(rows):
    grouped = defaultdict(list)
    for row in rows:
        grouped[row["pair_id"]].append(row)
    return grouped


# ============================================================
# API runner
# ============================================================

def make_client(base_url, api_key):
    return OpenAI(base_url=base_url, api_key=api_key)

def call_model(client, model, prompt, max_tokens=32, temperature=0.0, system_prompt=None):
    messages = []
    if system_prompt:
        messages.append({"role": "system", "content": system_prompt})
    messages.append({"role": "user", "content": prompt})

    resp = client.chat.completions.create(
        model=model,
        messages=messages,
        temperature=temperature,
        max_tokens=max_tokens,
    )
    return resp.choices[0].message.content.strip()

def run_predictions(
    client,
    model,
    dataset_rows,
    out_path,
    max_tokens=32,
    temperature=0.0,
    system_prompt="Answer as briefly as possible. Return only the final answer."
):
    preds = []
    for i, row in enumerate(dataset_rows, start=1):
        pred = call_model(
            client=client,
            model=model,
            prompt=row["prompt"],
            max_tokens=max_tokens,
            temperature=temperature,
            system_prompt=system_prompt,
        )
        preds.append({
            "id": row["id"],
            "prediction": pred,
        })

        if i % 25 == 0:
            print(f"  completed {i}/{len(dataset_rows)}")

    write_jsonl(out_path, preds)
    return preds


# ============================================================
# Paired evaluation
# ============================================================

def grade_exact(prediction, accepted_answers):
    pred = normalize_text(prediction)
    accepted = {normalize_text(a) for a in accepted_answers}
    return pred in accepted

def paired_evaluate(gold_rows, pred_rows):
    pred_map = {r["id"]: r.get("prediction", "") for r in pred_rows}
    detailed = []

    for gold in gold_rows:
        grader = gold["grader"]
        pred = pred_map.get(gold["id"], "")

        if grader["type"] != "exact_match_normalized":
            raise ValueError(f"Unsupported grader type: {grader['type']}")

        accepted = grader.get("accepted_answers", [gold["target"]])
        correct = int(grade_exact(pred, accepted))

        detailed.append({
            "id": gold["id"],
            "pair_id": gold["pair_id"],
            "base_item_id": gold.get("base_item_id"),
            "experiment": gold.get("experiment"),
            "variant": gold.get("variant"),
            "condition": gold.get("condition", {}),
            "target": gold.get("target"),
            "prediction": pred,
            "correct": correct,
            "missing_prediction": int(gold["id"] not in pred_map),
        })

    return detailed


# ============================================================
# Paired stats
# ============================================================

def mcnemar_counts(pairs, variant_a, variant_b):
    b = 0
    c = 0
    for _, pair_rows in pairs.items():
        by_variant = {r["variant"]: r for r in pair_rows}
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

def analyze_chunking(rows):
    pairs = group_by_pair(rows)
    deltas, flat_acc, chunked_acc = [], [], []

    for _, pair_rows in pairs.items():
        by_variant = {r["variant"]: r for r in pair_rows}
        if "flat" not in by_variant or "chunked" not in by_variant:
            continue
        flat = by_variant["flat"]["correct"]
        chunked = by_variant["chunked"]["correct"]
        flat_acc.append(flat)
        chunked_acc.append(chunked)
        deltas.append(chunked - flat)

    counts = mcnemar_counts(pairs, "chunked", "flat")

    return {
        "experiment": "chunking_v2",
        "n_pairs": len(deltas),
        "flat_accuracy": mean(flat_acc),
        "chunked_accuracy": mean(chunked_acc),
        "paired_gain": mean(deltas),
        "paired_gain_ci": bootstrap_ci(deltas),
        "mcnemar_counts": counts,
        "mcnemar_chi_square_cc": mcnemar_chi_square(counts),
    }

def analyze_interactivity(rows):
    pairs = group_by_pair(rows)
    deltas, low_acc, high_acc = [], [], []

    for _, pair_rows in pairs.items():
        by_variant = {r["variant"]: r for r in pair_rows}
        if "low" not in by_variant or "high" not in by_variant:
            continue
        low = by_variant["low"]["correct"]
        high = by_variant["high"]["correct"]
        low_acc.append(low)
        high_acc.append(high)
        deltas.append(low - high)

    counts = mcnemar_counts(pairs, "low", "high")

    return {
        "experiment": "element_interactivity_v2",
        "n_pairs": len(deltas),
        "low_accuracy": mean(low_acc),
        "high_accuracy": mean(high_acc),
        "paired_penalty": mean(deltas),
        "paired_penalty_ci": bootstrap_ci(deltas),
        "mcnemar_counts": counts,
        "mcnemar_chi_square_cc": mcnemar_chi_square(counts),
    }

def analyze_extraneous(rows):
    pairs = group_by_pair(rows)

    irrelevant_acc = []
    confusable_acc = []
    contradictory_acc = []

    delta_irrel_conf = []
    delta_irrel_contra = []
    delta_conf_contra = []

    valid_pairs = 0
    for _, pair_rows in pairs.items():
        by_variant = {r["variant"]: r for r in pair_rows}
        need = ["irrelevant", "confusable", "contradictory_adjacent"]
        if not all(v in by_variant for v in need):
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

    c1 = mcnemar_counts(pairs, "irrelevant", "confusable")
    c2 = mcnemar_counts(pairs, "irrelevant", "contradictory_adjacent")
    c3 = mcnemar_counts(pairs, "confusable", "contradictory_adjacent")

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
            "counts": c1,
            "chi_square_cc": mcnemar_chi_square(c1),
        },
        "mcnemar_irrelevant_vs_contradictory_adjacent": {
            "counts": c2,
            "chi_square_cc": mcnemar_chi_square(c2),
        },
        "mcnemar_confusable_vs_contradictory_adjacent": {
            "counts": c3,
            "chi_square_cc": mcnemar_chi_square(c3),
        },
    }

def analyze_paired(rows):
    exp = {r.get("experiment") for r in rows}
    if len(exp) != 1:
        raise ValueError(f"Expected one experiment, got {exp}")
    exp = next(iter(exp))

    if exp == "chunking_v2":
        return analyze_chunking(rows)
    elif exp == "element_interactivity_v2":
        return analyze_interactivity(rows)
    elif exp == "extraneous_load_v2":
        return analyze_extraneous(rows)
    else:
        raise ValueError(f"Unsupported experiment: {exp}")


# ============================================================
# Reporting
# ============================================================

def benchmark_sort_key(exp_name):
    order = {
        "chunking_v2": 0,
        "element_interactivity_v2": 1,
        "extraneous_load_v2": 2,
    }
    return order.get(exp_name, 999)

def summarize_model_effects(result):
    exp = result["experiment"]
    if exp == "chunking_v2":
        return result.get("paired_gain")
    if exp == "element_interactivity_v2":
        return result.get("paired_penalty")
    if exp == "extraneous_load_v2":
        return result.get("gap_irrelevant_minus_confusable")
    return None

def build_markdown_report(comparison):
    lines = ["# Paired V2 Multi-Model Report", ""]

    models = comparison["models"]
    datasets = sorted(comparison["datasets"], key=benchmark_sort_key)

    # Overall quick table
    headers = ["Model"] + datasets
    rows = []
    for model in models:
        row = [model]
        for ds in datasets:
            result = comparison["results"].get(model, {}).get(ds, {})
            value = summarize_model_effects(result)
            row.append(fmt(value))
        rows.append(row)

    lines.append("| " + " | ".join(headers) + " |")
    lines.append("|" + "|".join(["---"] * len(headers)) + "|")
    for row in rows:
        lines.append("| " + " | ".join(row) + " |")
    lines.append("")

    for ds in datasets:
        lines.append(f"## {ds}")
        lines.append("")
        for model in models:
            result = comparison["results"].get(model, {}).get(ds)
            if not result:
                continue

            lines.append(f"### {model}")
            lines.append("")

            if ds == "chunking_v2":
                ci = result["paired_gain_ci"]
                lines += [
                    f"- Flat accuracy: `{fmt(result['flat_accuracy'])}`",
                    f"- Chunked accuracy: `{fmt(result['chunked_accuracy'])}`",
                    f"- Paired gain: `{fmt(result['paired_gain'])}`",
                    f"- Bootstrap CI: `[{fmt(ci['ci_low'])}, {fmt(ci['ci_high'])}]`",
                    f"- McNemar counts: `{result['mcnemar_counts']}`",
                    f"- McNemar chi-square cc: `{fmt(result['mcnemar_chi_square_cc'])}`",
                    "",
                ]

            elif ds == "element_interactivity_v2":
                ci = result["paired_penalty_ci"]
                lines += [
                    f"- Low accuracy: `{fmt(result['low_accuracy'])}`",
                    f"- High accuracy: `{fmt(result['high_accuracy'])}`",
                    f"- Paired penalty: `{fmt(result['paired_penalty'])}`",
                    f"- Bootstrap CI: `[{fmt(ci['ci_low'])}, {fmt(ci['ci_high'])}]`",
                    f"- McNemar counts: `{result['mcnemar_counts']}`",
                    f"- McNemar chi-square cc: `{fmt(result['mcnemar_chi_square_cc'])}`",
                    "",
                ]

            elif ds == "extraneous_load_v2":
                ci1 = result["gap_irrelevant_minus_confusable_ci"]
                ci2 = result["gap_irrelevant_minus_contradictory_adjacent_ci"]
                ci3 = result["gap_confusable_minus_contradictory_adjacent_ci"]
                lines += [
                    f"- Irrelevant accuracy: `{fmt(result['irrelevant_accuracy'])}`",
                    f"- Confusable accuracy: `{fmt(result['confusable_accuracy'])}`",
                    f"- Contradictory adjacent accuracy: `{fmt(result['contradictory_adjacent_accuracy'])}`",
                    f"- Gap irrelevant-confusable: `{fmt(result['gap_irrelevant_minus_confusable'])}`",
                    f"- CI: `[{fmt(ci1['ci_low'])}, {fmt(ci1['ci_high'])}]`",
                    f"- Gap irrelevant-contradictory_adjacent: `{fmt(result['gap_irrelevant_minus_contradictory_adjacent'])}`",
                    f"- CI: `[{fmt(ci2['ci_low'])}, {fmt(ci2['ci_high'])}]`",
                    f"- Gap confusable-contradictory_adjacent: `{fmt(result['gap_confusable_minus_contradictory_adjacent'])}`",
                    f"- CI: `[{fmt(ci3['ci_low'])}, {fmt(ci3['ci_high'])}]`",
                    "",
                ]

    return "\n".join(lines)


# ============================================================
# Main orchestration
# ============================================================

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--datasets", nargs="+", required=True, help="List of paired v2 dataset JSONL files")
    parser.add_argument("--models", nargs="+", required=True, help="List of model names")
    parser.add_argument("--base-url", default=os.environ.get("OPENAI_BASE_URL", "https://api.openai.com/v1"))
    parser.add_argument("--api-key", default=os.environ.get("OPENAI_API_KEY"))
    parser.add_argument("--output-dir", default="paired_v2_runs")
    parser.add_argument("--max-tokens", type=int, default=32)
    parser.add_argument("--temperature", type=float, default=0.0)
    parser.add_argument("--system-prompt", default="Answer as briefly as possible. Return only the final answer.")
    args = parser.parse_args()

    if not args.api_key:
        raise ValueError("API key required via --api-key or OPENAI_API_KEY")

    ensure_dir(args.output_dir)
    client = make_client(args.base_url, args.api_key)

    comparison = {
        "models": args.models,
        "datasets": [],
        "results": {},
    }

    for model in args.models:
        model_slug = slugify(model)
        comparison["results"][model] = {}

        print(f"\n=== MODEL: {model} ===")

        for dataset_path in args.datasets:
            gold_rows = load_jsonl(dataset_path)
            exp_name = infer_experiment_name(dataset_path, gold_rows)

            if exp_name not in comparison["datasets"]:
                comparison["datasets"].append(exp_name)

            dataset_slug = slugify(exp_name)

            print(f"\nRunning {exp_name} on {model}")
            pred_path = os.path.join(args.output_dir, f"{model_slug}__{dataset_slug}__predictions.jsonl")
            eval_path = os.path.join(args.output_dir, f"{model_slug}__{dataset_slug}__paired_results.jsonl")
            analysis_path = os.path.join(args.output_dir, f"{model_slug}__{dataset_slug}__analysis.json")

            pred_rows = run_predictions(
                client=client,
                model=model,
                dataset_rows=gold_rows,
                out_path=pred_path,
                max_tokens=args.max_tokens,
                temperature=args.temperature,
                system_prompt=args.system_prompt,
            )

            detailed = paired_evaluate(gold_rows, pred_rows)
            write_jsonl(eval_path, detailed)

            analysis = analyze_paired(detailed)
            save_json(analysis_path, analysis)

            comparison["results"][model][exp_name] = analysis

            print(f"Saved predictions: {pred_path}")
            print(f"Saved paired results: {eval_path}")
            print(f"Saved analysis: {analysis_path}")

    comparison["datasets"] = sorted(comparison["datasets"], key=benchmark_sort_key)

    comparison_json = os.path.join(args.output_dir, "paired_comparison.json")
    comparison_md = os.path.join(args.output_dir, "paired_report.md")

    save_json(comparison_json, comparison)
    save_text(comparison_md, build_markdown_report(comparison))

    print(f"\nSaved comparison JSON to {comparison_json}")
    print(f"Saved markdown report to {comparison_md}")


if __name__ == "__main__":
    main()

Example usage
OpenAI-compatible endpoint
python paired_orchestrate_v2.py \
  --datasets chunking_v2_test.jsonl extraneous_load_v2_test.jsonl element_interactivity_v2_test.jsonl \
  --models gpt-4o-mini gpt-4.1-mini \
  --base-url https://api.openai.com/v1 \
  --api-key $OPENAI_API_KEY \
  --output-dir paired_v2_runs

RouteLLM
python paired_orchestrate_v2.py \
  --datasets chunking_v2_test.jsonl extraneous_load_v2_test.jsonl element_interactivity_v2_test.jsonl \
  --models route-llm \
  --base-url https://routellm.abacus.ai/v1 \
  --api-key $OPENAI_API_KEY \
  --output-dir paired_v2_runs


RouteLLM docs: API docs
Sample page: RouteLLM app

Output structure

Inside paired_v2_runs/ you’ll get files like:

gpt_4o_mini__chunking_v2__predictions.jsonl
gpt_4o_mini__chunking_v2__paired_results.jsonl
gpt_4o_mini__chunking_v2__analysis.json
gpt_4o_mini__extraneous_load_v2__analysis.json
paired_comparison.json
paired_report.md
How to read the report
Chunking

Positive paired gain means:

the same underlying item is easier when structured
Interactivity

Positive paired penalty means:

matched high-coupling items are harder than low-coupling ones
Extraneous load

Positive irrelevant-minus-confusable or irrelevant-minus-contradictory gaps mean:

not all extra tokens are equal
interference structure matters
Important caveat

This script is intentionally simple and useful.

It does not yet do:

resume/retry logic
rate limit handling
exact McNemar p-values
cross-model significance tests
per-condition subgroup analysis within each paired benchmark

Those are the best next upgrades.

Best next artifact

The strongest next addition would be a paired comparison plotter for paired_comparison.json that makes publication-ready figures for:

chunking gain by model
interactivity penalty by model
distractor sensitivity by model
confidence intervals across models

If you want, I can write that next.
