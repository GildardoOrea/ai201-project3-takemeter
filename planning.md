# TakeMeter — Planning Document

**Project:** AI201 Project 3 — TakeMeter
**Author:** Gildardo Orea Amador

---

## 1. Community

I chose three general gaming discussion communities on Reddit: **r/gaming**, **r/videogames**, and **r/Games**. I read these regularly, so I can tell the difference between a thoughtful post and a throwaway one.

These are a good fit for a discourse-quality classification task because the quality of posts varies enormously. r/Games skews toward substantive, reasoned discussion; r/gaming is dominated by memes, nostalgia, and quick emotional reactions; r/videogames sits in between. Pulling from all three gives me a wide spread of post quality, which is exactly what makes the classification task interesting — the same topic (say, a new release) produces a careful argument in one place and a one-line "this is peak 🔥" in another.

## 2. Labels

I use three mutually exclusive labels. The distinction is about *how* a take is made, not *what* it's about.

### `analysis`
A structured, evidence-based argument about a game, studio, or the industry. The post cites specific examples, mechanics, comparisons, or reasoning you could actually evaluate or push back on.
- *Example 1:* "Starfield's core problem is structural — loading-screen-gated 'exploration' removes the seamless traversal Bethesda games are built on, which is why it feels emptier than Skyrim despite having more raw content."
- *Example 2:* "Helldivers 2's monetization works because every premium item is also earnable through normal play. Compare that to Diablo 4 charging $28 for cosmetics with no in-game path — same business model, very different player trust."

### `hot_take`
A bold, confident opinion or judgment stated without real supporting evidence. The claim might be true, but the post asserts rather than argues.
- *Example 1:* "Single-player games are a dying breed and every publisher knows it."
- *Example 2:* "Nintendo hasn't made a genuinely good game since the Wii era."

### `reaction`
An immediate emotional or nostalgic response to a game, event, or piece of news — hype, excitement, grief, joy — with little to no argument.
- *Example 1:* "Just beat Elden Ring for the first time and I'm honestly tearing up, what a journey."
- *Example 2:* "BG3 winning Game of the Year made my whole year, so deserved."

## 3. Hard Edge Cases

The genuinely ambiguous posts sit between **analysis** and **hot_take**: a bold opinion that's *decorated* with one stat or example but isn't actually reasoning.

- *Example:* "Modern AAA games are a scam — Redfall launched at $70 and was broken on day one."

This has a confident judgment (hot_take) plus a specific example (looks like analysis).

**Decision rule:** If the evidence is specific and genuinely *supports* the claim through reasoning, label it `analysis`. If the "evidence" is one cherry-picked example used to justify a sweeping verdict, it's decoration — label it `hot_take`. The Redfall post uses one example to condemn all AAA games, so the reasoning doesn't hold → **hot_take**.

A second edge case sits between **reaction** and **hot_take**: an emotional post that also makes a claim ("This game is a MASTERPIECE, no notes!!"). **Decision rule:** if the post is primarily expressing a feeling in the moment with no argument, it's `reaction`; if it's stating an evaluative judgment as a position, it's `hot_take`.

## 4. Data Collection Plan

- **Source:** Top-level comments from discussion threads in r/gaming, r/videogames, and r/Games. I'm collecting **comments rather than image-post titles** because the comments contain real text to classify.
- **Target:** 200+ examples total, aiming for roughly **65–70 per label** to keep the distribution balanced (no label above ~70%).
- **Method:** Manual copy-paste into a spreadsheet (`text`, `label`, `notes` columns). I'll read each comment and apply the definitions above. I may use an LLM to pre-label a batch, then review and correct every single label myself (disclosed in the AI usage section).
- **If a label is underrepresented:** `analysis` is the hardest to find, so if I'm short on it after the first pass, I'll pull more from **r/Games** specifically, where substantive arguments concentrate, until each label clears ~20% of the dataset.

## 5. Evaluation Metrics

- **Overall accuracy** — a quick headline number, but not sufficient on its own because my classes won't be perfectly balanced and accuracy can hide failure on the minority class.
- **Per-class precision / recall / F1** — this is the metric I care about most. `analysis` is likely the rarest and hardest label, and I specifically want to know whether the model actually learned it or just leans on `hot_take`/`reaction`. F1 per class catches that.
- **Confusion matrix** — to see *which* boundary fails and in which direction. My hypothesis is the model will confuse `analysis` → `hot_take`, since that's the hardest human boundary too.

Accuracy alone isn't enough because a model that ignores `analysis` entirely could still post a decent accuracy if that class is small. Per-class F1 plus the confusion matrix are what reveal whether the model learned my actual taxonomy.

## 6. Definition of Success

- **Minimum bar:** the fine-tuned DistilBERT model **beats the zero-shot Groq baseline** on overall accuracy on the same test set.
- **Genuinely useful bar:** **per-class F1 ≥ 0.65 on all three labels** (not just the easy ones), and macro-F1 around **0.75+**. I care most about the `analysis` class clearing 0.65, because a tool that can't recognize substantive posts isn't useful for surfacing good discourse.
- **Deployment threshold:** for a real community tool (e.g., highlighting high-quality posts), I'd want macro-F1 ≥ 0.80 and the `analysis`/`hot_take` confusion under control, since mislabeling a rant as analysis is the most damaging error for that use case.

---

## AI Tool Plan

There's no application code to generate in this project, so my AI use focuses on the three places it actually helps:

- **Label stress-testing:** Before annotating, I'll give an LLM (Claude) my three label definitions and edge-case rules and ask it to generate 8–10 posts that sit on the `analysis`/`hot_take` boundary. If I can't classify its outputs cleanly with my own definitions, I'll tighten the definitions before labeling 200 real examples.
- **Annotation assistance:** I may use an LLM to pre-label a batch of collected comments to speed up annotation. I will review and correct **every** pre-assigned label against my own definitions — pre-labeling is a first pass, not the final label. This is disclosed in my README's AI usage section.
- **Failure analysis:** After fine-tuning, I'll paste the model's wrong predictions into an LLM and ask it to identify systematic patterns (e.g., short posts, sarcasm, a specific confused label pair). I'll then re-read those examples myself to verify each claimed pattern before putting it in my evaluation report.

*This document was written before data collection. It will be updated before starting any stretch features.*
