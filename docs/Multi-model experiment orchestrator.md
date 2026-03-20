Multi-model experiment orchestrator

Perfect. This is the missing piece that turns the benchmark into an actual research workflow.

This orchestrator will:

run multiple models
across multiple benchmark JSONL files
using an OpenAI-compatible API
save prediction files
evaluate them
write a single comparison summary JSON
print a compact leaderboard-style table

It does not depend on shell scripts. It’s a single Python entry point.

Assumptions

You already have:

generated benchmark datasets
the evaluation logic conceptually available

To keep this self-contained, I’ll include:

the API runner logic
the evaluation logic
the aggregation logic

So this can run standalone.

Install
pip install openai

Python script
import json
import time
import re
import argparse
from collections import defaultdict, Counter
from typing import Dict, Any, List

from openai import OpenAI

VOWELS = set("AEIOUaeiou")

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

def write_json(path: str, obj: Dict[str, Any]) -> None:
    with open(path, "w", encoding="utf-8") as f:
        json.dump(obj, f, indent=2, ensure_ascii=False)

# ============================================================
# Normalization
# ============================================================

def normalize_text(s: str) -> str:
    if s is None:
        return ""
    s = str(s).strip()
    s = re.sub(r"\s+", " ", s)
    return s

def normalize_for_exact(s: str) -> str:
    return normalize_text(s)

# ============================================================
# Constraint checker
# ============================================================

def is_vowel(c: str) -> bool:
    return c in VOWELS

def char_class(c: str) -> str:
    if c.isdigit():
        return "digit"
    if c.islower():
        return "lowercase"
    if c.isupper():
        return "uppercase"
    return "other"

def count_class(s: str, klass: str) -> int:
    if klass == "digit":
        return sum(ch.isdigit() for ch in s)
    if klass == "uppercase":
        return sum(ch.isupper() for ch in s)
    if klass == "lowercase":
        return sum(ch.islower() for ch in s)
    if klass == "vowel":
        return sum(is_vowel(ch) for ch in s)
    raise ValueError(f"Unknown class {klass}")

def evaluate_constraint(pred: str, rule: Dict[str, Any]) -> bool:
    t = rule["type"]

    if t == "position":
        idx = rule["index"]
        return len(pred) > idx and pred[idx] == rule["value"]

    if t == "must_include":
        return rule["value"] in pred

    if t == "all_distinct":
        return len(set(pred)) == len(pred)

    if t == "count_class":
        return count_class(pred, rule["class"]) == rule["count"]

    if t == "before":
        a = pred.find(rule["a"])
        b = pred.find(rule["b"])
        return a != -1 and b != -1 and a < b

    if t == "not_adjacent_classes":
        for i in range(len(pred) - 1):
            c1 = char_class(pred[i])
            c2 = char_class(pred[i + 1])
            if (c1 == rule["class_a"] and c2 == rule["class_b"]) or \
               (c1 == rule["class_b"] and c2 == rule["class_a"]):
                return False
        return True

    return False

def evaluate_constraint_set(pred: str, constraints: List[Dict[str, Any]]) -> Dict[str, Any]:
    pred = normalize_text(pred)
    passed = [evaluate_constraint(pred, rule) for rule in constraints]
    total = len(constraints)
    satisfied = sum(passed)
    return {
        "exact": satisfied == total,
        "satisfied": satisfied,
        "total": total,
        "per_constraint": passed,
    }

# ============================================================
# Exact grading
# ============================================================

def evaluate_exact(pred: str, accepted_answers: List[str]) -> bool:
    pred_n = normalize_for_exact(pred)
    accepted_n = {normalize_for_exact(a) for a in accepted_answers}
    return pred_n in accepted_n

# ============================================================
# API runner
# ============================================================

def build_messages(prompt: str) -> List[Dict[str, str]]:
    return [
        {
            "role": "system",
            "content": "You are a precise benchmark model. Answer as briefly as possible and follow formatting instructions exactly."
        },
        {
            "role": "user",
            "content": prompt
        }
    ]

def call_model(client: OpenAI, model: str, prompt: str, temperature: float, max_tokens: int, retries: int = 3) -> Dict[str, Any]:
    last_err = None
    for attempt in range(retries):
        try:
            resp = client.chat.completions.create(
                model=model,
                messages=build_messages(prompt),
                temperature=temperature,
                max_tokens=max_tokens,
            )
            text = resp.choices[0].message.content if resp.choices else ""
            usage = getattr(resp, "usage", None)
            return {
                "prediction": text.strip() if text else "",
                "raw_output": text if text else "",
                "usage": {
                    "prompt_tokens": getattr(usage, "prompt_tokens", None) if usage else None,
                    "completion_tokens": getattr(usage, "completion_tokens", None) if usage else None,
                    "total_tokens": getattr(usage, "total_tokens", None) if usage else None,
                },
            }
        except Exception as e:
            last_err = str(e)
            time.sleep(1.5 * (attempt + 1))
    return {
        "prediction": "",
        "raw_output": "",
        "error": last_err,
        "usage": {},
    }

def run_predictions(
    client: OpenAI,
    model: str,
    dataset_rows: List[Dict[str, Any]],
    temperature: float,
    max_tokens: int,
    sleep_seconds: float = 0.0,
    limit: int = None,
) -> List[Dict[str, Any]]:
    rows = dataset_rows[:limit] if limit is not None else dataset_rows
    preds = []

    for i, row in enumerate(rows, start=1):
        result = call_model(client, model, row["prompt"], temperature, max_tokens)
        out = {
            "id": row["id"],
            "prediction": result.get("prediction", ""),
            "raw_output": result.get("raw_output", ""),
            "model": model,
            "temperature": temperature,
            "usage": result.get("usage", {}),
        }
        if "error" in result:
            out["error"] = result["error"]

        preds.append(out)
        print(f"[{model}] {i}/{len(rows)} {row['id']} -> {repr(out['prediction'][:60])}")

        if sleep_seconds > 0:
            time.sleep(sleep_seconds)

    return preds

# ============================================================
# Evaluation
# ============================================================

def predictions_to_map(pred_rows: List[Dict[str, Any]]) -> Dict[str, str]:
    return {r["id"]: r.get("prediction", "") for r in pred_rows}

def condition_key(condition: Dict[str, Any]) -> str:
    return "|".join(f"{k}={condition[k]}" for k in sorted(condition.keys()))

def evaluate_dataset(gold_rows: List[Dict[str, Any]], pred_rows: List[Dict[str, Any]]) -> Dict[str, Any]:
    pred_map = predictions_to_map(pred_rows)

    total = 0
    exact_correct = 0
    css_scores = []
    by_condition = defaultdict(lambda: {"n": 0, "exact": 0, "css_sum": 0.0})
    errors = 0

    for row in gold_rows:
        total += 1
        rid = row["id"]
        pred = pred_map.get(rid, "")
        cond = condition_key(row.get("condition", {}))
        by_condition[cond]["n"] += 1

        if pred == "":
            errors += 1

        grader = row["grader"]
        if grader["type"] == "exact_match_normalized":
            accepted = grader.get("accepted_answers", [row["target"]])
            ok = evaluate_exact(pred, accepted)
            if ok:
                exact_correct += 1
                by_condition[cond]["exact"] += 1

        elif grader["type"] == "constraint_checker":
            ev = evaluate_constraint_set(pred, grader["constraints"])
            css = ev["satisfied"] / ev["total"] if ev["total"] else 0.0
            css_scores.append(css)
            by_condition[cond]["css_sum"] += css
            if ev["exact"]:
                exact_correct += 1
                by_condition[cond]["exact"] += 1

        else:
            raise ValueError(f"Unknown grader type {grader['type']}")

    condition_summary = {}
    for ck, stats in by_condition.items():
        n = stats["n"]
        condition_summary[ck] = {
            "n": n,
            "exact_accuracy": stats["exact"] / n if n else 0.0,
            "mean_constraint_satisfaction": stats["css_sum"] / n if n else 0.0,
        }

    return {
        "n_examples": total,
        "empty_or_missing_predictions": errors,
        "exact_accuracy": exact_correct / total if total else 0.0,
        "mean_constraint_satisfaction": sum(css_scores) / len(css_scores) if css_scores else None,
        "by_condition": condition_summary,
    }

# ============================================================
# Aggregation
# ============================================================

def benchmark_short_name(path: str) -> str:
    name = path.split("/")[-1].split("\\")[-1]
    if name.endswith(".jsonl"):
        name = name[:-6]
    return name

def aggregate_experiment_results(all_results: Dict[str, Dict[str, Dict[str, Any]]]) -> Dict[str, Any]:
    """
    all_results[model_name][benchmark_name] = summary
    """
    leaderboard = []

    for model_name, benchmarks in all_results.items():
        exacts = []
        csss = []

        for bench_name, summary in benchmarks.items():
            exacts.append(summary["exact_accuracy"])
            if summary["mean_constraint_satisfaction"] is not None:
                csss.append(summary["mean_constraint_satisfaction"])

        macro_exact = sum(exacts) / len(exacts) if exacts else 0.0
        macro_css = sum(csss) / len(csss) if csss else None

        row = {
            "model": model_name,
            "macro_exact_accuracy": macro_exact,
            "macro_constraint_satisfaction": macro_css,
            "benchmarks": benchmarks,
        }
        leaderboard.append(row)

    leaderboard.sort(key=lambda x: x["macro_exact_accuracy"], reverse=True)

    return {
        "leaderboard": leaderboard
    }

def print_leaderboard(summary: Dict[str, Any]) -> None:
    print("\n=== Leaderboard ===")
    for i, row in enumerate(summary["leaderboard"], start=1):
        macro_css = row["macro_constraint_satisfaction"]
        css_str = f"{macro_css:.4f}" if macro_css is not None else "n/a"
        print(
            f"{i:>2}. {row['model']:<25} "
            f"macro_exact={row['macro_exact_accuracy']:.4f} "
            f"macro_css={css_str}"
        )
        for bench_name, bench in row["benchmarks"].items():
            mcss = bench["mean_constraint_satisfaction"]
            mcss_str = f"{mcss:.4f}" if mcss is not None else "n/a"
            print(
                f"    - {bench_name:<35} "
                f"exact={bench['exact_accuracy']:.4f} "
                f"css={mcss_str}"
            )

# ============================================================
# Main
# ============================================================

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--benchmarks", nargs="+", required=True, help="Benchmark JSONL files")
    parser.add_argument("--models", nargs="+", required=True, help="Model names")
    parser.add_argument("--api-key", required=True, help="API key")
    parser.add_argument("--base-url", default=None, help="OpenAI-compatible base URL")
    parser.add_argument("--temperature", type=float, default=0.0)
    parser.add_argument("--max-tokens", type=int, default=32)
    parser.add_argument("--sleep", type=float, default=0.0)
    parser.add_argument("--limit", type=int, default=None)
    parser.add_argument("--save-preds", action="store_true", help="Save raw prediction files")
    parser.add_argument("--pred-prefix", default="preds", help="Prefix for prediction output files")
    parser.add_argument("--out", default="benchmark_comparison.json", help="Output summary JSON")
    args = parser.parse_args()

    client_kwargs = {"api_key": args.api_key}
    if args.base_url:
        client_kwargs["base_url"] = args.base_url
    client = OpenAI(**client_kwargs)

    benchmark_data = {benchmark_short_name(path): load_jsonl(path) for path in args.benchmarks}
    all_results = {}

    for model_name in args.models:
        print(f"\n### Running model: {model_name}")
        model_results = {}

        for bench_name, rows in benchmark_data.items():
            print(f"\n## Benchmark: {bench_name}")
            pred_rows = run_predictions(
                client=client,
                model=model_name,
                dataset_rows=rows,
                temperature=args.temperature,
                max_tokens=args.max_tokens,
                sleep_seconds=args.sleep,
                limit=args.limit,
            )

            if args.save_preds:
                pred_path = f"{args.pred_prefix}_{model_name}_{bench_name}.jsonl".replace("/", "_")
                write_jsonl(pred_path, pred_rows)

            eval_summary = evaluate_dataset(rows[:args.limit] if args.limit else rows, pred_rows)
            model_results[bench_name] = eval_summary

            print(
                f"[{model_name} | {bench_name}] "
                f"exact={eval_summary['exact_accuracy']:.4f} "
                f"css={eval_summary['mean_constraint_satisfaction'] if eval_summary['mean_constraint_satisfaction'] is not None else 'n/a'}"
            )

        all_results[model_name] = model_results

    final_summary = aggregate_experiment_results(all_results)
    write_json(args.out, final_summary)
    print_leaderboard(final_summary)
    print(f"\nSaved comparison summary to {args.out}")

if __name__ == "__main__":
    main()

Example usage
Run 2 models across 4 benchmarks
python orchestrate_benchmarks.py \
  --benchmarks constraint_stacking_test.jsonl extraneous_load_test.jsonl chunking_test.jsonl element_interactivity_test.jsonl \
  --models route-llm gpt-4o-mini \
  --api-key YOUR_API_KEY \
  --base-url https://routellm.abacus.ai/v1 \
  --temperature 0 \
  --max-tokens 32 \
  --save-preds \
  --out comparison.json

Important note about multiple providers

If the models you want to compare live on different base URLs / providers, then this script should be run separately per provider, or upgraded to support per-model endpoint mapping.

Right now it assumes:

one base URL
one API key
many model names on that same endpoint

That’s often fine for OpenAI-compatible routers.

Output structure

The comparison.json file contains:

per-model summaries
per-benchmark exact accuracy
per-benchmark mean constraint satisfaction
macro exact accuracy across benchmarks
What the leaderboard tells you

This gives you an immediate way to see:

which model is most robust overall
which one collapses on constraint stacking
which one is most distractor-sensitive
whether chunking helps one class of models more than another
whether interactivity penalties differ by model

That gets very close to your original thesis: that models differ in how they handle load.

Best next upgrade

The most valuable upgrade now is probably:

Option 1: Multi-model comparison plotting

Take comparison.json and plot:

one line per model
one benchmark per subplot
accuracy vs load parameter

This would make “capacity-sensitive overload” visually obvious.

Option 2: Paired-condition significance testing

Especially for:

chunked vs flat
irrelevant vs confusable distractors
low vs high interactivity
Option 3: Research memo

Turn the whole workflow into a clear hypothesis + methods + expected-results document.

If you want, I’d do Option 1 next: a comparison plotter for comparison.json.
