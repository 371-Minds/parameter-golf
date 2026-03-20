Each benchmark example is stored as JSONL.

Example schema:

{
  "id": "cs_test_000001",
  "experiment": "constraint_stacking",
  "prompt": "Produce a 6-character string that satisfies the following constraints: ...",
  "target": "A7bcDe",
  "condition": {
    "num_constraints": 4,
    "output_length": 6
  },
  "grader": {
    "type": "constraint_checker",
    "constraints": [
      {"type": "position", "index": 0, "value": "A"},
      {"type": "count_class", "class": "digit", "count": 1}
    ]
  }
}

Prediction file format:

{"id": "cs_test_000001", "prediction": "A7bcDe"}

