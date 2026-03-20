Yes — I think cognitive load theory is a very useful lens here, even if we have to translate it carefully from humans to transformers.

And for parameter golf, it may actually be one of the best conceptual lenses.

Short answer

Your intuition is strong:

humans fail when working memory gets overloaded
tiny LLMs fail when representational bandwidth and optimization capacity get overloaded
both degrade first on tasks requiring too many simultaneously active dependencies

So in a tiny-model challenge, you can think of the whole problem as:

Minimize unnecessary representational load while maximizing useful compression

That is extremely close to cognitive load theory.

Teardown: parameter golf through a cognitive load lens

Classic cognitive load theory splits load into:

intrinsic load — inherent complexity of the material
extraneous load — complexity caused by poor presentation
germane load — effort spent building useful schemas

That maps surprisingly well to tiny language models.

1. Intrinsic load → irreducible statistical complexity of language

This is the part the model must learn:

syntax
token dependencies
morphology
discourse regularities
common factual co-occurrence structure
code/text mixed patterns if present in corpus

For a tiny model, intrinsic load is brutal because language has high entropy and long-range structure.

So the model is already capacity-stressed before bad design choices even start.

2. Extraneous load → wasted capacity from architecture or pipeline inefficiency

This is where I think your point gets sharp.

A small LLM absolutely can experience something analogous to overload when its limited capacity is spent on the wrong things:

bad tokenization
poor data ordering
unstable optimization
over-deep architecture that can’t train in time
oversized embeddings
unnecessary attention heads
serialization overhead
positional scheme mismatch
noisy or unhelpful training distribution
poor batch/learning-rate regime

That is basically extraneous cognitive load for a model.

Not because it “feels strain,” but because its finite optimization budget is consumed by nonproductive structure.

3. Germane load → formation of reusable latent schemas

This is the good stuff:

learning common syntax templates
recurring semantic frames
punctuation and formatting priors
word-piece compositionality
common internet-text structures
reusable statistical abstractions

A tiny model wins when it spends almost all available capacity on these high-reuse latent schemas.

That’s why compression-based evaluation is such a good fit: it directly rewards internalization of reusable regularities.

Do LLMs “experience cognitive load”?

Strictly speaking: not phenomenologically, no.

But functionally? Yes, in an analogous sense.

I’d phrase it like this:

LLM cognitive load is the pressure on finite active representational and optimization resources caused by the amount, dispersion, and interaction complexity of information that must be encoded or coordinated at once.

That shows up as:

attention dilution
context interference
gradient competition
shallow heuristic collapse
failure on compositional tasks
recency bias
brittle long-range dependency tracking
degradation under too many simultaneous constraints

So yes — not “load” as subjective effort, but load as capacity stress in information processing.

That’s very real.

Why this matters especially for tiny models

Large models can brute-force around some inefficiencies.

Tiny models cannot.

In parameter golf, every bad design choice becomes equivalent to overloading working memory.

A tiny model only has so much room for:

token statistics
syntax priors
long-range structure
rare words
formatting conventions
world regularities

If you force it to juggle too many unrelated burdens, it fragments.

That fragmentation is the model analogue of overload.

Practical teardown: how I’d design for low cognitive load

If I were attacking this challenge using your frame, I’d optimize around three principles:

A. Reduce extraneous load

Meaning: remove all architectural and training waste.

Likely moves
tie input/output embeddings
keep architecture simple and highly optimized
avoid over-parameterized attention for the size regime
pick width/depth ratio that converges fast
avoid fancy mechanisms unless they clearly improve early learning
use highly stable normalization choices
minimize artifact overhead
use aggressive but stable LR schedule
ensure data pipeline is dead simple and fast

Question for every component:

Is this helping the model learn reusable structure, or is it consuming scarce representational budget?

B. Match intrinsic load to capacity

Don’t ask a tiny model to absorb the whole universe equally.

In practice this means:

choose data ordering that starts with easier/high-frequency patterns
maybe curriculum-like exposure if rules allow
front-load dense, reusable linguistic structure
ensure the model masters common patterns before rare tail complexity

That’s classic load management: don’t swamp the learner before schemas form.

C. Maximize schema formation

This is the hidden win condition.

You want the model to learn abstractions that compress many cases at once:

punctuation patterns
HTML/common web text structures
repeated phrase templates
function word transitions
local syntax scaffolds
common discourse markers
spelling and morphology regularities

A tiny model lives or dies by reusability density.

Architectural implications

If we take cognitive load seriously, then model design should emphasize:

1. Efficient routing of limited capacity

A tiny model can’t afford diffuse attention everywhere.

So you’d want:

fewer but more meaningful heads
maybe wider FFN relative to attention if it improves reusable pattern storage
balanced depth so information is transformed enough without becoming optimization-fragile
2. Low-interference representations

One of the biggest forms of “load” in small models is interference: new learning overwrites or competes with existing useful patterns.

So I’d care a lot about:

normalization placement
optimizer choice
learning rate warmup/decay
token frequency balancing
avoiding too much rare-token distraction early
3. Fast schema bootstrapping

Since training time is capped at 10 minutes, you don’t just need asymptotic quality. You need rapid acquisition of compressive schemas.

That suggests:

simple, highly trainable transformer blocks
maybe muP-inspired scaling instincts
extremely tuned initialization and LR
data stream designed for immediate structure pickup
Where tiny LLMs visibly show “cognitive overload”

You can often see it in failure modes like:

losing track of nested syntax
collapsing under too many simultaneous constraints
confusing near-neighbor concepts
overusing short-range heuristics
failing to integrate earlier context with later qualifiers
spiking error on rare-but-compositional sequences
becoming brittle when many features must be jointly satisfied

Those are all analogous to overload phenomena: too many active dependencies, not enough coordination budget.

A more formal framing

You could define model cognitive load as a function of:

L=f(Ci​,Cx​,Cg​,R,T,O)
Where:
Ci​
= intrinsic complexity of the input distribution
Cx​
= extraneous burden from architecture/training/interface
Cg​
= useful schema-building effort
R
= representational capacity
T
= available training time / compute budget
O

= optimization efficiency

Then parameter golf becomes:

minbits-per-bytesubject toartifact≤16MB, train time≤10 min

while implicitly trying to minimize:

R⋅OCx​​

and maximize:

TCg​​

In plain English:

waste as little capacity as possible
turn learning effort into reusable abstractions quickly
What I’d test if I were serious about this theory

If we wanted to operationalize your idea, I’d run ablations around “load management”:

Data-side
curriculum vs shuffled data
high-frequency-first exposure
short-context-first then longer-context
cleaner text first vs mixed noise
Model-side
width vs depth tradeoff under equal parameter budget
fewer heads vs more heads
FFN-heavy vs attention-heavy ratios
tied vs untied embeddings
RoPE vs simpler positional schemes
Training-side
warmup length
token budget allocation
optimizer stability under tiny training windows
gradient clipping thresholds
label smoothing or none
quantization-aware vs post-hoc compression

And I would specifically interpret outcomes as:

which settings reduce interference?
which accelerate schema formation?
which waste capacity on extraneous burdens?
My actual opinion

I think you are onto something important.

Not in the fluffy sense of “models are just like humans,” but in the stronger systems sense that:

both humans and LLMs are finite-capacity inference systems that break when coordination demands exceed efficient internal organization.

That’s a real shared principle.

The mechanisms differ:

humans: biological working memory, attentional control, schema acquisition
LLMs: attention bandwidth, parameter allocation, gradient dynamics, representation interference

But the structural phenomenon is similar enough to be useful.

And in small-model design, it’s especially useful because the whole game is about:

preventing overload
prioritizing reusable patterns
reducing waste
sequencing learning

Which is basically cognitive load engineering.

My blunt strategic takeaway

If someone approaches parameter golf like:

“How do I cram a transformer into 16MB?”

they’ll do okay.

If someone approaches it like:

“How do I design a learner whose tiny capacity is protected from extraneous load and rapidly converted into reusable schemas?”

they may do much better.

That second framing is much closer to how winners in constrained systems competitions usually think.

If you want, I can go one step deeper and do either:
A concrete winning architecture sketch for this challenge
A formal “LLM cognitive load theory” framework you could use as a research lens
A set of experiments to prove tiny LLMs exhibit overload effects similar to humans
