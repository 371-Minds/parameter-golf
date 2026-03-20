Research memo skeleton: Cognitive Load Theory for Tiny LLMs

Below is a concise but serious memo structure you can use as the basis for:

a note
internal research doc
workshop paper seed
benchmark README with theory

I’ll write it in a form that’s halfway between a research abstract and a concept note.

Title

Do Tiny Language Models Exhibit Cognitive Load Effects?
A Benchmarking Framework for Capacity Stress, Interference, and Structure Sensitivity in Small LLMs

One-sentence thesis

Tiny language models appear to show structured performance collapse under increasing dependency and interference load, and this behavior can be studied using a cognitive load theory lens.

Abstract

We propose that very small language models can be usefully analyzed as bounded-capacity symbolic-statistical processors whose failures are not merely random error, but often reflect systematic overload phenomena analogous to cognitive load effects in humans. We distinguish between intrinsic load arising from the number and interdependence of task elements, extraneous load arising from irrelevant or misleading presentation structure, and schema-supporting structure that reduces effective load through chunking and organization. To test this claim, we introduce a suite of synthetic, auto-gradable benchmarks targeting four axes: constraint stacking, extraneous distraction, chunking benefits, and element interactivity. We evaluate models using exact accuracy, constraint satisfaction, condition-wise overload curves, and structured failure taxonomies. We hypothesize that smaller models will exhibit earlier and steeper collapse as task load increases; that distractors will impair performance disproportionately when they are confusable rather than merely irrelevant; that chunked presentation will improve accuracy without changing underlying content; and that highly interactive facts will be more difficult than equally numerous independent facts. If observed, these patterns would support the view that tiny LLM failures reflect organized capacity limitations rather than generic noise, with implications for architecture design, evaluation, compression, and parameter-efficient intelligence.

1. Motivation

Large language models are often discussed in terms of scale, emergent capabilities, and generalization. But at the low-parameter end, models frequently display a different regime: brittle reasoning, rapid degradation under multitask constraints, sensitivity to formatting, and failure to maintain multiple active dependencies at once.

These phenomena suggest a useful analogy: tiny LLMs may behave like capacity-limited processors subject to cognitive load constraints.

This framing matters because it gives us:

a vocabulary for model failure
a benchmark design principle
a way to distinguish raw knowledge deficits from coordination deficits
a possible theory of why presentation and structure matter so much in small models

Rather than asking only whether a model knows something, we ask: how much active structure can it coordinate before performance collapses?

2. Core claim

We do not claim that LLMs literally experience human cognition or conscious effort.

We do claim that they may exhibit an analogous computational pattern:

limited active representational bandwidth
interference between competing elements
dependence on structural organization
threshold-like collapse under coupled demands

This makes cognitive load theory a useful descriptive and experimental lens, even if the underlying mechanism is not psychologically identical.

3. Theoretical mapping

We map classical cognitive load categories to small-model behavior as follows.

3.1 Intrinsic load

Intrinsic load corresponds to the amount of task-relevant structure that must be simultaneously coordinated.

For LLMs, this includes:

number of active constraints
number of facts that must be jointly integrated
dependency depth
order sensitivity
cross-reference requirements

A task with many independent facts may be easier than a task with fewer but tightly coupled facts.

3.2 Extraneous load

Extraneous load corresponds to processing burden imposed by representation rather than by the task itself.

For LLMs, this includes:

distractor statements
misleading adjacency
confusable alternatives
poor formatting
split attention across dispersed information
instruction clutter

If model performance drops when irrelevant content is added without changing the underlying problem, that is evidence of structure-sensitive overload.

3.3 Germane load / schema support

Rather than treating “germane load” as literal effort, we reinterpret it as schema-supporting organization.

For LLMs, this includes:

chunking
templated structure
grouped facts
explicit intermediate organization
decomposed prompts

If the same latent information becomes easier when structured well, that suggests organization reduces effective load.

4. Hypotheses
H1. Constraint overload

As the number of simultaneously active constraints increases, tiny model exact accuracy will decline sharply, often faster than local constraint satisfaction.

H2. Interference-sensitive distraction

Performance will decline more under confusable or contradictory distractors than under equally numerous but clearly irrelevant distractors.

H3. Chunking benefit

Models will perform better when equivalent information is presented in chunked, grouped, or structured form than when presented as unstructured flat text.

H4. Element interactivity penalty

Tasks requiring integration across highly interdependent facts will be harder than tasks with the same number of independent facts.

H5. Capacity-scaling effect

Smaller models will exhibit earlier and steeper overload curves than larger models on all four dimensions.

5. Benchmark design

We propose four synthetic benchmark families.

5.1 Constraint stacking

The model must produce an output satisfying multiple simultaneous rules, such as:

character in position
must include symbol
digit count
order constraint
uniqueness constraint

This isolates coordination burden.

5.2 Extraneous load

The model must answer a simple question from a brief context, while the context includes distractors varying in:

count
type
similarity to target
adjacency to relevant facts

This isolates interference burden.

5.3 Chunking

The same underlying information is presented either:

as flat text
or in grouped/chunked form

The question remains identical.

This isolates representational organization effects.

5.4 Element interactivity

The model answers questions based on either:

independent facts
or facts that must be linked through multi-step dependency chains

This isolates dependency coupling.

6. Measurements

Primary metrics:

exact accuracy
mean constraint satisfaction
condition-wise accuracy slopes

Secondary metrics:

overload slope with increasing task load
gap between local and global success
performance delta between paired task forms
failure-type frequencies

Failure taxonomy examples:

omission of earlier constraints
recency bias toward later constraints
distractor substitution
partial integration failure
format confusion
duplication failure
local success with global inconsistency
7. Predicted signature of overload

If the cognitive load framing is correct, we expect the following pattern:

increasing active constraints reduces exactness before it eliminates all local skill
confusable distractors hurt more than irrelevant distractors
chunking improves performance without changing information content
high interactivity hurts more than increased fact count alone
smaller models show steeper and earlier collapse

The key point is that failure should be structured, not random.

8. Why this matters

This framework matters for both science and engineering.

Scientific relevance

It offers:

a mechanistic lens for tiny model failure
a bridge between cognitive science concepts and model behavior
a way to talk about “reasoning brittleness” more precisely
Engineering relevance

It helps with:

prompt design for small models
architecture choices for compressed systems
evaluation of edge models
parameter-efficient model development
“parameter golf” style competitions where capacity allocation matters

In highly compressed models, reducing extraneous processing burden may be as important as increasing raw capability.

9. Relation to compression and parameter golf

The parameter golf setting is especially relevant because models must allocate extremely scarce representational resources.

Under these conditions:

every extra burden matters
inefficient representation becomes fatal
architecture must minimize waste
tokenization and prompt structure become first-class concerns

From this perspective, parameter golf is not just about smaller models. It is about: the most efficient allocation of limited cognitive bandwidth.

10. Limitations

This framework has important limitations.

The analogy to human cognition is functional, not literal.
Synthetic tasks may not fully capture naturalistic reasoning.
Prompting differences can confound apparent load sensitivity.
Some failures may reflect training-data mismatch rather than overload.
Larger models may use memorized heuristics that mimic structure sensitivity.

So the framework should be treated as:

a disciplined experimental lens
not a proof of human-like cognition
11. Future work

Natural next steps include:

paired-condition benchmarks
confidence and calibration analysis
logprob-based overload markers
hidden-state analysis for representational crowding
training interventions that explicitly improve load robustness
architecture comparisons under equal parameter budgets
evaluating whether curriculum and chunked supervision improve resilience
12. Conclusion

Tiny LLMs may not merely fail because they “know less.”
They may fail because they can coordinate less, resist interference less, and benefit more from structure.

A cognitive load lens gives us a principled way to study those limits.

If these overload patterns are empirically confirmed, then the design of small language models should focus not only on parameter count and loss, but on:

coordination efficiency
interference resistance
structural economy
schema-supporting representation

That would make cognitive load theory not just a metaphor for LLMs, but a practical framework for building better small models.

Short version for README / project front page

You could condense the whole thing to this:

This project tests whether tiny language models exhibit structured overload effects analogous to cognitive load phenomena in humans. We evaluate four dimensions: constraint stacking, extraneous distraction, chunking benefit, and element interactivity. The core hypothesis is that small models fail not only because they know less, but because they can coordinate fewer active dependencies, are more vulnerable to interference, and depend more heavily on structural organization.

Strong next move

If you want, I can now turn this into one of three concrete artifacts:

a polished workshop-style paper draft
a GitHub README with benchmark framing and usage
an experimental results template with tables/figures placeholders

The highest leverage next step is probably 3, because it lets you drop in real results immediately once you run the suite.
