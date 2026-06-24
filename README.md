# TakeMeter — Classifying Discourse Quality in r/assassinscreed

**Author:** Gildardo Orea Amador
**Course:** AI201 — Project 3

TakeMeter is a fine-tuned text classifier that sorts comments from the Assassin's Creed subreddit into three kinds of "takes": substantive **analysis**, evidence-free **hot takes**, and emotional **reactions**. I collected and hand-labeled 206 real comments, fine-tuned DistilBERT on them, and compared it against a zero-shot Groq (llama-3.3-70b) baseline.

The short version: **the zero-shot baseline (74%) beat my fine-tuned model (48%)**, and digging into *why* turned out to be the most interesting part of the project.

---

## Community

I chose **r/assassinscreed**. I follow the sub, so I can tell a thoughtful comment from a throwaway one, and the discourse quality varies enormously — the same topic (a new release, a patch, the modern-day storyline) produces a carefully reasoned argument in one comment and a one-line "this is peak " in the next. That spread is exactly what makes the classification task interesting: it's not about *what* people are talking about, it's about *how* they're making their point.

## Label Taxonomy

Three mutually exclusive labels, focused on *how* a take is made rather than its topic:

### `analysis`
A structured, evidence-based argument about a game, studio, or the industry that cites specific examples, mechanics, comparisons, or reasoning you could evaluate.
- *"Ghost did everything Valhalla did but better. AC's story is 100+ hours but bloated with filler, while Ghost is 30 hours of refined content."*
- *"Desmond was the face of the games that had quality modern day, and the AC1–3 games had an actual plot running in tandem with the historical plot. That's why people note the stark difference between AC1–3 and AC4–present."*

### `hot_take`
A bold, confident opinion or judgment stated without real supporting evidence.
- *"Single-player gaming is a dying breed and every publisher knows it."*
- *"Origins is better than Odyssey."*

### `reaction`
An immediate emotional or nostalgic response with little to no argument.
- *"Just beat Elden Ring for the first time and I'm honestly tearing up, what a journey."*
- *"That foliage. God damn."*

## Data Collection & Annotation

- **Source:** Top-level comments from public discussion threads in r/assassinscreed (peak-of-the-series debates, the modern-day/Layla story, Ubisoft pricing/microtransactions, Ghost of Tsushima comparisons, fan art, and AC: Shadows screenshot reactions). I collected comments rather than image-post titles so there was real text to classify.
- **Process:** I copied raw comment blocks, stripped the usernames/vote counts/ads/off-topic banter, and labeled each comment against the definitions in my `planning.md`. I used an LLM to pre-label batches and reviewed the labels myself; genuinely borderline cases are flagged in the `notes` column of `dataset.csv` (see the AI Usage section for full disclosure).
- **Size & balance:** 206 examples, almost perfectly balanced (no single label above 34%, well under the 70% imbalance limit):

| Label | Count | % |
|---|---|---|
| analysis | 69 | 33% |
| hot_take | 69 | 33% |
| reaction | 68 | 33% |

**Split:** 70% train / 15% validation / 15% test (144 / 31 / 31), stratified by label.

### Three genuinely difficult examples
1. *"I really hate the fact they included rams in this game. Rams didn't exist at the time. And they change the whole look of the ship."* — Sits between `hot_take` and `analysis`: it's an angry complaint, but it has one real factual point (historical accuracy). **Decision:** the fact is decorative, used to justify a gut reaction rather than build an argument → `hot_take`.
2. *"Imma be real: why do we treat Darby McDevitt in such high regard when the modern day has never been that good... it's so non-committal and abstract that it feels like a placeholder."* — Heated and opinionated, but it actually argues a case with reasoning. **Decision:** there's a real argument under the heat → `analysis`.
3. *"Domain mode made this update a net zero to me... Domain mode as a whole fucking SUCKS. Especially locking what little MD content we have behind it."* — Strong verdict ("net zero", "SUCKS") plus a concrete reason (locking content). **Decision:** the emotional verdict dominates the one supporting reason → `hot_take`.

## Fine-Tuning Approach

- **Base model:** `distilbert-base-uncased` with a 3-class sequence-classification head.
- **Training setup:** 3 epochs, learning rate 2e-5, batch size 16, weight decay 0.01, 50 warmup steps, on a free Colab T4 GPU. Best model selected by validation accuracy.
- **Hyperparameter decision:** I kept the default **3 epochs**, reasoning that more epochs risk overfitting on only 144 training examples. In hindsight this may have been the wrong call — the model's confidence scores never rose above ~0.37 (barely above the 0.33 random floor for 3 classes), which points to **under**fitting rather than overfitting. More epochs or a higher learning rate would be the first thing I'd try next.

## Baseline

A zero-shot baseline using Groq's **`llama-3.3-70b-versatile`** with no task-specific training. I wrote a system prompt that names the community, defines all three labels in plain language with one example each, and instructs the model to output only the label name. Each of the 31 test comments was sent individually at `temperature=0`, and the model's reply was matched back to a label string (all 31 responses parsed cleanly).

---

## Evaluation Report

### Overall accuracy

| Model | Accuracy | Macro F1 |
|---|---|---|
| Zero-shot baseline (Groq llama-3.3-70b) | **0.742** | 0.68 |
| Fine-tuned DistilBERT | **0.484** | 0.44 |
| **Difference** | **−0.258** | −0.24 |

Random guessing on a balanced 3-class task is ~0.33. The fine-tuned model barely clears that; the baseline is comfortably above it. **Fine-tuning was a regression of ~26 points.**

### Per-class metrics

**Fine-tuned DistilBERT**
| Label | Precision | Recall | F1 |
|---|---|---|---|
| analysis | 0.45 | 0.82 | 0.58 |
| hot_take | 0.67 | 0.20 | 0.31 |
| reaction | 0.50 | 0.40 | 0.44 |

**Zero-shot baseline (Groq)**
| Label | Precision | Recall | F1 |
|---|---|---|---|
| analysis | 0.92 | 1.00 | 0.96 |
| hot_take | 1.00 | 0.20 | 0.33 |
| reaction | 0.59 | 1.00 | 0.74 |

**The single most important number in this whole project:** *both* models have `hot_take` recall of **0.20**. Neither model — small or huge — can recognize a hot take. The difference is only in where the misses go.

### Confusion matrix — Fine-tuned model (test set)

Rows = true label, columns = predicted label. (Image: `confusion_matrix.png`.)

| true ↓ / pred → | analysis | hot_take | reaction |
|---|---|---|---|
| **analysis** | 9 | 0 | 2 |
| **hot_take** | 6 | 2 | 2 |
| **reaction** | 5 | 1 | 4 |

The fine-tuned model predicted `analysis` for **20 of 31** comments. It collapsed the whole task into "everything is analysis," which is why analysis recall is high (0.82) but its precision is terrible (0.45). The baseline does the opposite with its misses — it pushes the 8 missed hot takes mostly into `reaction` instead.

### Three wrong predictions, analyzed

1. **"Biggest problem is you have two different studios making the games."** — True `hot_take`, predicted `analysis` (conf 0.34). The phrasing *sounds* analytical ("biggest problem", a named cause), so the model treated declarative, topic-heavy structure as an argument — even though there's zero evidence, just an asserted opinion. This is the core `hot_take → analysis` collapse.
2. **"Valhalla would suck if it wasn't for how good it looks, but that world is so incredibly gorgeous it's insane."** — True `hot_take`, predicted `reaction` (conf 0.34). Here the emotional intensifiers ("suck", "gorgeous", "insane") dragged it toward `reaction`. The model keys on sentiment words and misses that there's an underlying evaluative judgment.
3. **"I'm so glad you can customize hair... because let's face it, Vikings have always been shown with really sick hairstyles."** — True `reaction`, predicted `analysis` (conf 0.34). The "because..." justification clause makes an excited personal reaction *look* like a reasoned argument, so the model read explanatory structure as analysis.

**The pattern:** the model latches onto surface features — declarative phrasing reads as `analysis`, strong emotion words read as `reaction` — and never learned the actual distinction, which is whether real evidence backs the claim. Since I labeled these consistently, this is a model/data-size problem, not an annotation problem: 144 examples isn't enough to teach that boundary.

### Sample classifications (fine-tuned model)

| Comment (truncated) | Predicted | Confidence | True | ✓ |
|---|---|---|---|---|
| "Fuck layla, this fucking modern times is boring as shit." | hot_take | 0.34 | hot_take | ✓ |
| "Something I really love about hers is the inside of the hood being white." | reaction | 0.34 | reaction | ✓ |
| "You can still make an awesome outfit from sets and have the matching sets look as cohesive as any outfit from Origins..." | analysis | 0.35 | analysis | ✓ |
| "Biggest problem is you have two different studios making the games." | analysis | 0.34 | hot_take | ✗ |
| "I'm so glad you can customize hair... Vikings have always been shown with really sick hairstyles." | analysis | 0.34 | reaction | ✗ |

The first correct one is a clean call: *"Fuck layla, this fucking modern times is boring as shit"* is pure asserted opinion with no evidence, and the model labeled it `hot_take` — reasonable, even if the 0.34 confidence shows it wasn't sure. That low confidence is the real story: **every prediction, right or wrong, sits at 0.34–0.37.** The model is essentially guessing with a slight lean on every single example.

## Reflection: what the model learned vs. what I intended

I intended the model to learn a distinction about **evidence** — does the comment back its claim with specific reasoning, or just assert/emote? What it actually learned was a much shallower thing: a weak association between **surface vocabulary** and labels. Long, declarative, topic-heavy comments get called `analysis`; comments full of emotion words get called `reaction`; and `hot_take` — which has neither a distinctive vocabulary nor a clean structural signal — gets almost nothing, because it's defined by what it *lacks* (evidence) rather than what it contains. You can't learn an absence from 144 examples.

The near-uniform ~0.34 confidences confirm the model never built separable internal representations of the three classes. The zero-shot 70b model does better not because it was "trained" on this, but because it already understands the *concepts* of an argument vs. an assertion vs. an emotional reaction from general pretraining — something a small model can't pick up from a couple hundred examples. The honest conclusion is that this task is genuinely hard, and for a subjective, evidence-based distinction, a large instruction-tuned model is the better tool unless I can label far more data.

## Spec Reflection

- **One way the spec helped:** Writing `planning.md` *before* collecting data forced me to name the hardest edge case in advance — the `analysis ↔ hot_take` boundary. When the results came in, that's *exactly* where both models failed. Having predicted it made the failure analysis sharp instead of a surprise.
- **One way my implementation diverged:** My `planning.md` "definition of success" was that the fine-tuned model would beat the baseline and hit ≥0.65 F1 per class. It did neither (0.484 accuracy, hot_take F1 0.31). I left that criterion as written instead of softening it after the fact, because the gap between what I expected and what happened is the most useful result in the project.

## AI Usage

I used an LLM (Claude) at three points, and reviewed/edited its output each time:

1. **Taxonomy design & planning.** I directed it to help refine my 3-label taxonomy and draft `planning.md`. I chose the community (r/assassinscreed) and made the final call on the labels and the edge-case decision rules; I edited the document to match my own reasoning.
2. **Annotation assistance (disclosed).** I directed it to extract the actual comment text from the raw Reddit blocks I collected and pre-label each comment using my definitions. I reviewed the labels, and the borderline cases it flagged are recorded in the `notes` column of `dataset.csv`. The community choice, label definitions, and final dataset were mine.
3. **Failure analysis & write-up.** After running the notebook, I had it help interpret the confusion matrix and draft this README. I provided all the real numbers and edited the analysis; the interpretation of *why* the baseline beat fine-tuning reflects my own reading of the results.

---

## Video Walkthrough

<!-- Add your 3–5 minute demo link/GIF here -->
_Demo video: (to be added)_

## Files in this repo
- `planning.md` — design doc (labels, edge cases, metrics, AI plan), written before data collection
- `dataset.csv` — 206 labeled comments (`text`, `label`, `notes`)
- `confusion_matrix.png` — fine-tuned model confusion matrix
- `evaluation_results.json` — accuracy summary for both models
- `README.md` — this report
