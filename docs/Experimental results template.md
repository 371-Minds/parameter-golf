Experimental Results Template

## Title

**Do Tiny Language Models Exhibit Cognitive Load Effects? Experimental Results**

## 1. Experimental Setup

### 1.1 Models
We evaluated the following models:

- Model A:
- Model B:
- Model C:

For each model, report:
- parameter count if known
- provider / endpoint
- decoding settings
- context length if relevant

### 1.2 Decoding configuration

- Temperature:
- Max tokens:
- System prompt:
- Number of trials per item:
- Deterministic or stochastic decoding:

### 1.3 Benchmarks

We evaluated four benchmark families:

- Constraint Stacking
- Extraneous Load
- Chunking
- Element Interactivity

For each benchmark, specify:
- number of train / val / test items
- condition ranges
- whether tasks were paired across conditions

### 1.4 Metrics

Primary metrics:
- Exact Accuracy
- Mean Constraint Satisfaction

Secondary metrics:
- Overload slope
- Distractor sensitivity
- Chunking gain
- Interactivity penalty
- Failure type frequencies

## 2. Main Results

### 2.1 Overall leaderboard

| Model | Constraint Stacking | Extraneous Load | Chunking | Element Interactivity | Macro Exact | Macro CSS |
|------|----------------------|-----------------|----------|-----------------------|-------------|-----------|
| Model A |  |  |  |  |  |  |
| Model B |  |  |  |  |  |  |
| Model C |  |  |  |  |  |  |

### 2.2 Key findings

Summarize the most important observations:

- 
- 
- 

## 3. Constraint Stacking Results

### 3.1 Accuracy by number of constraints

| Model | 2 Constraints | 3 Constraints | 4 Constraints | 5 Constraints | 6 Constraints |
|------|----------------|---------------|---------------|---------------|---------------|
| Model A |  |  |  |  |  |
| Model B |  |  |  |  |  |
| Model C |  |  |  |  |  |

### 3.2 Mean constraint satisfaction

| Model | 2 Constraints | 3 Constraints | 4 Constraints | 5 Constraints | 6 Constraints |
|------|----------------|---------------|---------------|---------------|---------------|
| Model A |  |  |  |  |  |
| Model B |  |  |  |  |  |
| Model C |  |  |  |  |  |

### 3.3 Interpretation

Questions to answer:
- Does exact accuracy fall faster than partial constraint satisfaction?
- Is there a threshold where performance collapses?
- Do smaller models show sharper decline?

### 3.4 Failure taxonomy

| Failure Type | Model A | Model B | Model C |
|-------------|---------|---------|---------|
| Constraint omission |  |  |  |
| Recency override |  |  |  |
| Position bias |  |  |  |
| Global inconsistency |  |  |  |
| Duplication failure |  |  |  |
| Count mismatch |  |  |  |

### 3.5 Narrative

Example interpretation:

> As the number of simultaneous constraints increased, exact accuracy declined sharply for all models, while mean constraint satisfaction decreased more gradually. This suggests that local rule compliance remains partially intact even after global coordination begins to fail, consistent with a bounded-capacity overload account.

## 4. Extraneous Load Results

### 4.1 Accuracy by distractor count and type

| Model | Irrelevant (Low) | Irrelevant (High) | Confusable (Low) | Confusable (High) | Contradictory Adjacent |
|------|-------------------|-------------------|------------------|-------------------|------------------------|
| Model A |  |  |  |  |  |
| Model B |  |  |  |  |  |
| Model C |  |  |  |  |  |

### 4.2 Interpretation

Questions to answer:
- Do confusable distractors hurt more than irrelevant distractors?
- Is the performance drop explained by length alone?
- Does one model resist interference better than others?

### 4.3 Narrative

Example interpretation:

> Performance degradation was strongly dependent on distractor type. Confusable and contradictory distractors impaired accuracy much more than equally numerous irrelevant distractors, indicating that interference, not merely prompt length, drives much of the observed failure.

## 5. Chunking Results

### 5.1 Flat vs chunked accuracy

| Model | Flat | Chunked | Gain |
|------|------|---------|------|
| Model A |  |  |  |
| Model B |  |  |  |
| Model C |  |  |  |

### 5.2 By query type

| Model | Query Type | Flat | Chunked | Gain |
|------|------------|------|---------|------|
| Model A | lookup |  |  |  |
| Model A | comparison |  |  |  |
| Model B | lookup |  |  |  |
| Model B | comparison |  |  |  |

### 5.3 Interpretation

Questions to answer:
- Does chunking help consistently?
- Is the benefit larger for smaller models?
- Does structure matter more on comparison than lookup tasks?

### 5.4 Narrative

Example interpretation:

> Chunked presentation improved performance across nearly all models and task types, despite holding latent information constant. This supports the view that representational organization reduces effective processing burden.

## 6. Element Interactivity Results

### 6.1 Accuracy by fact count and interactivity

| Model | Low-Interactivity 3 Facts | Low-Interactivity 5 Facts | High-Interactivity 3 Facts | High-Interactivity 5 Facts |
|------|----------------------------|---------------------------|----------------------------|----------------------------|
| Model A |  |  |  |  |
| Model B |  |  |  |  |
| Model C |  |  |  |  |

### 6.2 Interpretation

Questions to answer:
- Are tightly coupled facts harder than equally numerous independent facts?
- Does the gap widen as fact count increases?
- Is there evidence of multi-hop coordination failure?

### 6.3 Narrative

Example interpretation:

> High-interactivity tasks were consistently harder than low-interactivity tasks with the same number of facts. The gap widened as fact count increased, suggesting that dependency coupling imposes a distinct burden beyond raw informational quantity.

## 7. Cross-Benchmark Synthesis

### 7.1 Summary table

| Model | Overload Sensitivity | Interference Sensitivity | Chunking Benefit | Interactivity Penalty |
|------|-----------------------|--------------------------|------------------|-----------------------|
| Model A |  |  |  |  |
| Model B |  |  |  |  |
| Model C |  |  |  |  |

### 7.2 Main synthesis points

Suggested prompts:
- Which model is most robust overall?
- Which model degrades most sharply under active coordination burden?
- Which benchmark most clearly shows structure-sensitive overload?
- Are the effects consistent with the cognitive load framing?

### 7.3 Narrative

Example interpretation:

> Across all four benchmarks, smaller models showed steeper degradation under increasing task load, greater vulnerability to confusable distractors, and larger gains from chunked presentation. These converging effects support the hypothesis that tiny LLM failures are structured by bounded coordination capacity and interference sensitivity.

## 8. Threats to Validity

Potential caveats:
- benchmark artifacts
- decoding sensitivity
- prompt-template bias
- training contamination
- synthetic-to-natural transfer limits

## 9. Conclusion

Template text:

> The results provide preliminary evidence that tiny language models exhibit structured overload behavior across multiple task families. These effects are consistent with a cognitive load interpretation in which failure arises not only from missing knowledge, but from limited active coordination capacity, interference susceptibility, and sensitivity to representational structure.

## 10. Figures to include

- Figure 1: Macro exact accuracy by model
- Figure 2: Constraint stacking overload curves
- Figure 3: Extraneous load by distractor type
- Figure 4: Flat vs chunked comparison
- Figure 5: Element interactivity curves

## 11. Appendix

Suggested appendix materials:
- prompt templates
- grader definitions
- generation parameters
- full condition tables
- additional failure examples
Best next step
The most useful immediate artifact now would be:

a filled-in results compiler that reads comparison.json and auto-generates markdown tables for this template
a paper-style draft using the memo + these sections
a README badge/table generator for quick repo presentation
The best practical next move is 1. I can write a script that converts comparison.json directly into markdown tables you can paste into the README or results document.
