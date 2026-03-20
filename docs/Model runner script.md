Model runner script for OpenAI-compatible APIs

Here’s a practical runner that:

reads benchmark JSONL
sends each prompt to an OpenAI-compatible API
writes predictions JSONL
works for chat-completions style endpoints
supports custom base URLs
is suitable for RouteLLM/OpenAI-compatible providers

It uses the official openai Python SDK interface style.

What you need installed
pip install openai

Python script
import json
import time
import argparse
from typing import List, Dict, Any

from openai import OpenAI

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

def append_jsonl(path: str, row: Dict[str, Any]) -> None:
    with open(path, "a", encoding="utf-8") as f:
        f.write(json.dumps(row, ensure_ascii=False) + "\n")

# ============================================================
# Generation
# ============================================================

def build_messages(prompt: str) -> List[Dict[str, str]]:
    return [
        {
            "role": "system",
            "content": "You are a precise benchmark model. Answer as briefly as possible and follow the user's formatting instructions exactly."
        },
        {
            "role": "user",
            "content": prompt
        }
    ]

def call_model(
    client: OpenAI,
    model: str,
    prompt: str,
    temperature: float = 0.0,
    max_tokens: int = 64,
    timeout_retries: int = 3,
    sleep_seconds: float = 1.0,
) -> Dict[str, Any]:
    last_err = None
    for attempt in range(timeout_retries):
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
                "raw_response": text if text else "",
                "usage": {
                    "prompt_tokens": getattr(usage, "prompt_tokens", None) if usage else None,
                    "completion_tokens": getattr(usage, "completion_tokens", None) if usage else None,
                    "total_tokens": getattr(usage, "total_tokens", None) if usage else None,
                }
            }
        except Exception as e:
            last_err = str(e)
            time.sleep(sleep_seconds * (attempt + 1))

    return {
        "prediction": "",
        "raw_response": "",
        "error": last_err,
        "usage": {
            "prompt_tokens": None,
            "completion_tokens": None,
            "total_tokens": None,
        }
    }

# ============================================================
# Resume support
# ============================================================

def load_existing_ids(path: str) -> set:
    try:
        rows = load_jsonl(path)
        return {r["id"] for r in rows if "id" in r}
    except FileNotFoundError:
        return set()

# ============================================================
# Main runner
# ============================================================

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", required=True, help="Benchmark dataset JSONL")
    parser.add_argument("--output", required=True, help="Predictions JSONL output")
    parser.add_argument("--model", required=True, help="Model name")
    parser.add_argument("--api-key", required=True, help="API key")
    parser.add_argument("--base-url", default=None, help="OpenAI-compatible base URL")
    parser.add_argument("--temperature", type=float, default=0.0)
    parser.add_argument("--max-tokens", type=int, default=64)
    parser.add_argument("--limit", type=int, default=None, help="Optional max rows to run")
    parser.add_argument("--resume", action="store_true", help="Resume and skip completed ids")
    parser.add_argument("--sleep", type=float, default=0.0, help="Sleep between requests")
    args = parser.parse_args()

    client_kwargs = {"api_key": args.api_key}
    if args.base_url:
        client_kwargs["base_url"] = args.base_url
    client = OpenAI(**client_kwargs)

    rows = load_jsonl(args.input)
    if args.limit is not None:
        rows = rows[:args.limit]

    done_ids = load_existing_ids(args.output) if args.resume else set()

    total = len(rows)
    processed = 0

    for i, row in enumerate(rows, start=1):
        rid = row["id"]
        if rid in done_ids:
            continue

        prompt = row["prompt"]
        result = call_model(
            client=client,
            model=args.model,
            prompt=prompt,
            temperature=args.temperature,
            max_tokens=args.max_tokens,
        )

        out_row = {
            "id": rid,
            "prediction": result.get("prediction", ""),
            "raw_output": result.get("raw_response", ""),
            "model": args.model,
            "temperature": args.temperature,
            "usage": result.get("usage", {}),
        }
        if "error" in result:
            out_row["error"] = result["error"]

        append_jsonl(args.output, out_row)
        processed += 1

        print(f"[{i}/{total}] {rid} -> {repr(out_row['prediction'][:80])}")

        if args.sleep > 0:
            time.sleep(args.sleep)

    print(f"Done. Wrote {processed} new predictions to {args.output}")

if __name__ == "__main__":
    main()

Example usage
OpenAI-compatible generic endpoint
python run_benchmark.py \
  --input constraint_stacking_test.jsonl \
  --output cs_preds.jsonl \
  --model your-model-name \
  --api-key YOUR_API_KEY \
  --base-url https://your-openai-compatible-endpoint/v1 \
  --temperature 0 \
  --max-tokens 32

With resume support
python run_benchmark.py \
  --input extraneous_load_test.jsonl \
  --output el_preds.jsonl \
  --model your-model-name \
  --api-key YOUR_API_KEY \
  --base-url https://your-openai-compatible-endpoint/v1 \
  --resume

If you want to use RouteLLM-style OpenAI-compatible APIs

The system info here says the RouteLLM API base URL is:

API docs

Base URL: https://routellm.abacus.ai/v1

So your command would look like:

python run_benchmark.py \
  --input chunking_test.jsonl \
  --output ch_preds.jsonl \
  --model route-llm \
  --api-key YOUR_API_KEY \
  --base-url https://routellm.abacus.ai/v1 \
  --temperature 0 \
  --max-tokens 32

Full workflow
1. Generate datasets
python generate_datasets.py

2. Run model
python run_benchmark.py \
  --input constraint_stacking_test.jsonl \
  --output cs_preds.jsonl \
  --model route-llm \
  --api-key YOUR_API_KEY \
  --base-url https://routellm.abacus.ai/v1

3. Evaluate
python evaluate.py \
  --gold constraint_stacking_test.jsonl \
  --pred cs_preds.jsonl \
  --out cs_summary.json

4. Plot
python plot_overload.py --cs cs_summary.json --failures

Good defaults

For these tasks, I’d recommend:

temperature=0
max_tokens=16 to 32 for most tasks
max_tokens=48 for constraint stacking if the model rambles
a strict system prompt, which the script already uses
Important caveat

Different APIs may vary a bit in:

field compatibility
support for max_tokens
token accounting
error format

If one endpoint is slightly different, the script may need a tiny adaptation.

Most useful next upgrade

At this point, the highest-value next artifact is probably one of:

a bash/Makefile pipeline for generate → run → evaluate → plot
a multi-model experiment orchestrator that runs several models and compares them
a paper-style methodology/results memo
a paired-example v2 benchmark generator

If your aim is to move fast experimentally, I’d recommend 2 next: a multi-model orchestrator that runs the whole benchmark suite across several models and outputs a comparison table.
