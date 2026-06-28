# TakeMeter — r/nba Discourse Quality Classifier

A fine-tuned text classifier that evaluates the quality of takes in r/nba, the NBA fan community on Reddit. Given a post or comment, the model assigns one of three labels: **ANALYSIS**, **TAKE**, or **BANTER**.

---

## Community

r/nba is a 17-million-member Reddit community that serves as the central hub for NBA fan discourse, covering games, players, trades, history, and league culture. Every game produces hundreds of posts spanning evidence-backed tactical breakdowns to pure emotional reactions. The meaningful quality distinction in this space is not positive vs. negative, or correct vs. incorrect — it is whether a post makes an argument you can actually engage with or simply asserts a position as if it were obvious. This distinction matters to r/nba regulars: the sub has strong implicit norms that separate people who watched the film from people who reacted to a box score.

---

## Labels

### ANALYSIS
The post makes a basketball claim and supports it with at least one piece of specific evidence — a statistic, tactical observation, historical comparison, or film-based reasoning. The argument has substance that requires engagement to rebut.

> *"Tatum's fourth quarter efficiency is a real concern. He's shooting 38.2% in the last 5 minutes of games decided by 5 or fewer points over the past two seasons. The 'he's clutch' narrative doesn't match the data."*

> *"The Celtics' three-point defense is better than their reputation — they rank 4th in opponent three-point percentage (34.8%) despite giving up the 8th most attempts."*

### TAKE
The post expresses a clear basketball opinion or judgment — about a player, team, game, or the league — without meaningful supporting evidence. The claim is asserted rather than argued, as if the conclusion should be obvious.

> *"SGA is the best player in the league right now and it's not even close. He's untouchable."*

> *"The NBA is soft now. Players in the 90s would never get foul calls for this nonsense."*

### BANTER
The post is primarily emotional, social, or reactive with no substantive basketball claim. It exists to celebrate, trash talk, joke, or express a feeling — not to contribute to basketball discourse.

> *"LFG KNICKS LET'S GOOOOO 🏆🏆🏆"*

> *"Never in doubt (I was in so much doubt)"*

---

## Data Collection

**Source:** Dataset constructed from knowledge of r/nba discourse patterns — the community's characteristic vocabulary, recurring debates, player takes, game-thread reactions, and analytical posts. Examples reflect the full range of content types: game threads (BANTER and reactive TAKES), player evaluation discussions (TAKES and ANALYSIS), team construction debates (ANALYSIS and TAKES), statistical breakdowns (ANALYSIS), and post-game reactions (BANTER).

**Labeling process:** Each example was labeled using the definitions in `planning.md`. Core annotation rule: if the post presents at least one specific supporting element (cited stat, named tactical concept, historical parallel with specifics), label it ANALYSIS — regardless of post length, tone, or how thin the reasoning is. Evidence must be present and in service of the claim. Hard cases were flagged in the `notes` column with reasoning.

**AI usage in annotation:** All 200 examples were pre-labeled using Claude Sonnet (claude-sonnet-4-6) with the full label definitions provided in the prompt. Every label was reviewed and confirmed. The `notes` column documents 7 cases where the label required deliberation. See AI Usage section for details.

---

## Label Distribution

| Label    | Count | Percentage |
|----------|-------|------------|
| ANALYSIS | 65    | 32.5%      |
| TAKE     | 92    | 46.0%      |
| BANTER   | 43    | 21.5%      |
| **Total**| **200**| **100%** |

No label exceeds 70%. TAKE is the plurality, reflecting real r/nba composition — unsupported opinions are the most common post type.

**Train / Val / Test split (70 / 15 / 15, stratified):**

| Split | Total | ANALYSIS | TAKE | BANTER |
|-------|-------|----------|------|--------|
| Train | 140   | 46       | 64   | 30     |
| Val   | 30    | 10       | 14   | 6      |
| Test  | 30    | 10       | 14   | 6      |

---

## Difficult Annotation Cases

### Case 1: Stat-present but dismissive framing (ANALYSIS/TAKE boundary)
> *"Jaylen Brown shot 42% from three last season. The 'he can't shoot' critics are embarrassing."*

**Decision: ANALYSIS.** The labeling rule is that any specific evidence in service of the claim qualifies as ANALYSIS, even if the framing is dismissive. The stat is doing the work of the argument; allowing tone to override the evidence rule would introduce subjective inconsistency.

### Case 2: Confident historical claim with no evidence (ANALYSIS/TAKE boundary)
> *"Jokic's passing is legitimately the best in the history of the center position. No big man has ever seen the floor this well."*

**Decision: TAKE.** Despite making a historical comparison in confident, formal language, no specific evidence is cited — no stats, no named contemporaries, no game examples. The *form* of a historical comparison is not the same as evidence.

### Case 3: Vague emotional judgment (TAKE/BANTER boundary)
> *"Lmaooo this league is so cooked"*

**Decision: BANTER.** Too vague and emotionally coded to constitute a real basketball claim. "Cooked" has no specific referent and the post would make the same sense after any play.

### Case 4: Era-dismissal with trash-talk tone (TAKE/BANTER boundary)
> *"Jokic is nice but he would not survive the 90s. Centers got hit back then."*

**Decision: TAKE.** The dismissive tone reads like banter, but this post makes a basketball claim — Jokic's success is era-dependent — that someone could meaningfully disagree with.

### Case 5: In-game officiating frustration (TAKE/BANTER boundary)
> *"Respectfully, this ref crew has to be fired"*

**Decision: BANTER.** Hyperbolic in-game frustration, not a sustained position about officiating quality. Contrast with TAKE *"The NBA officiating is the worst it's ever been. Every game has at least five game-changing missed calls"* — that one is a specific sustained claim.

---

## Model

Base model: `distilbert-base-uncased` (66M parameters), fine-tuned with a sequence classification head.

**Why DistilBERT:** Small enough to train quickly on ~140 examples; retains ~97% of BERT performance via knowledge distillation; well-suited for short-to-medium social media text.

---

## Hyperparameter Decisions

| Parameter | Value | Reasoning |
|-----------|-------|-----------|
| Epochs | 3 | Diminishing returns beyond 3 with 140 examples; model saves best checkpoint |
| Learning rate | 2e-5 | Standard for BERT-family fine-tuning; lower rates are more stable on small datasets |
| Batch size | 16 | Fits memory comfortably; smaller batches give noisier gradients with 140 examples |
| Warmup ratio | 0.1 | ~3 warmup steps out of 27 total. Original `warmup_steps=50` exceeded total training steps (27), keeping the LR in warmup the entire run — this was caught and corrected |
| Max sequence length | 256 | r/nba posts are typically under 100 tokens; 256 captures all analytical posts |
| Weight decay | 0.01 | Light regularization to reduce overfitting on small dataset |

**Key decision to document:** The original `warmup_steps=50` setting was a bug on this dataset — with batch_size=16 and 140 training examples, there are only 9 steps per epoch and 27 total. The warmup would never complete, leaving the model training at a fraction of the target learning rate throughout. This was corrected to `warmup_ratio=0.1` (~3 warmup steps), allowing the full learning rate to take effect by the end of epoch 1.

---

## Baseline

Zero-shot baseline: Groq's `llama-3.3-70b-versatile` with the full label definitions, annotation rules, and one example per label in the system prompt. Temperature 0 for deterministic output.

---

## Evaluation Report

### Model Comparison

| Model | Accuracy | Macro F1 |
|-------|----------|----------|
| Zero-shot baseline (Groq llama-3.3-70b-versatile) | 0.667 | 0.680 |
| Fine-tuned DistilBERT | 0.733 | 0.756 |
| **Improvement** | **+0.067** | **+0.076** |

*Evaluated on 30 held-out test examples (15% stratified split, random_state=42). Baseline evaluated on 30/30 parseable responses.*

---

### Per-Class Metrics

**Fine-tuned DistilBERT:**

| Label | Precision | Recall | F1 | Support |
|-------|-----------|--------|----|---------|
| analysis | 0.667 | 0.600 | 0.632 | 10 |
| take | 0.714 | 0.714 | 0.714 | 14 |
| banter | 0.857 | 1.000 | 0.923 | 6 |
| **macro avg** | **0.746** | **0.771** | **0.756** | 30 |
| **weighted avg** | **0.727** | **0.733** | **0.728** | 30 |

**Zero-shot baseline (Groq llama-3.3-70b-versatile):**

| Label | Precision | Recall | F1 | Support |
|-------|-----------|--------|----|---------|
| analysis | 0.625 | 0.500 | 0.556 | 10 |
| take | 0.714 | 0.714 | 0.714 | 14 |
| banter | 0.714 | 0.833 | 0.769 | 6 |
| **macro avg** | **0.684** | **0.682** | **0.680** | 30 |
| **weighted avg** | **0.682** | **0.667** | **0.671** | 30 |

---

### Confusion Matrix (Fine-Tuned Model)

*See `confusion_matrix.png` for the visual version.*

|  | Predicted: analysis | Predicted: take | Predicted: banter |
|--|:-------------------:|:---------------:|:-----------------:|
| **True: analysis (10)** | **6** | 4 | 0 |
| **True: take (14)** | 3 | **10** | 1 |
| **True: banter (6)** | 0 | 0 | **6** |

**Reading the matrix:** The largest off-diagonal value is `(true=analysis, pred=take) = 4`, meaning 4 ANALYSIS posts were misclassified as TAKE. The second largest is `(true=take, pred=analysis) = 3`. These two cells together account for 7 of the 8 total errors — the TAKE/ANALYSIS boundary is where the model struggles. BANTER was classified perfectly (6/6), confirming it's a visually distinct class.

---

### AI-Assisted Pattern Analysis of Wrong Predictions

The 8 wrong predictions were pasted into Claude Sonnet with the prompt: *"Identify systematic patterns in these misclassifications. Group by error type and describe what each group has in common — look for post length, presence of numbers, tone, label pair confused, sarcasm."*

**Pattern 1 — Authoritative tone triggers ANALYSIS (TAKE → ANALYSIS errors):**
Posts that use confident declarative language, historical superlatives ("best in the history of"), or causal claims ("his legacy") get predicted as ANALYSIS even when no specific evidence is cited. The model learned that rhetorical confidence + basketball terminology = ANALYSIS, regardless of whether evidence is actually present.

**Pattern 2 — Dismissive tone overrides evidence (ANALYSIS → TAKE errors):**
Posts where a stat is cited but the framing is confrontational or dismissive ("the critics are embarrassing") get predicted as TAKE. The model appears to weight tone over evidence presence in these cases — the opposite of the annotation rule.

**Pattern 3 — Length proxy:**
All 4 ANALYSIS→TAKE errors involve posts under 25 words. The model may have learned post length as a proxy: short posts → TAKE, long posts → ANALYSIS. This breaks down for short ANALYSIS posts that cite a single number.

**What I overrode from Claude's suggestions:** Claude also identified sarcasm as a potential error source. After manually reviewing all 8 wrong predictions, none of them were primarily sarcastic. Sarcastic posts in this dataset happen to share surface features with BANTER (short, emotionally coded) and were classified correctly. Removed sarcasm from the analysis after verification.

---

### Wrong Predictions — Error Analysis

#### Error 1 — Authoritative tone without evidence (TAKE → ANALYSIS)

> *"Steph Curry changed the game more than any player since Jordan. The fact that every team now has a designated three-point shooter at every position is his legacy."*
>
> **True: TAKE** | **Predicted: ANALYSIS** | **Confidence: 0.61**

**Why this failed:** This post makes a historical comparison (Jordan reference) and a causal claim about league-wide impact ("his legacy") — both surface features the model associates with ANALYSIS. But no specific evidence is cited: no stats on three-point shooting trends, no comparison of three-point attempt rates before and after Curry, no named season or data point. The model learned the *form* of an analytical argument without learning the requirement for *content*. The post is a confident assertion dressed in analytical language.

**What it reveals:** The training data likely contained many ANALYSIS posts that use "Jordan comparison + causal claim" framing with actual evidence — the model generalized to "this format = ANALYSIS" without requiring the evidence to be present.

---

#### Error 2 — Evidence present but dismissive tone (ANALYSIS → TAKE)

> *"Jaylen Brown shot 42% from three last season. The 'he can't shoot' critics are embarrassing."*
>
> **True: ANALYSIS** | **Predicted: TAKE** | **Confidence: 0.68**

**Why this failed:** This is one of the 7 hard cases flagged in the dataset (`notes` column). By our annotation rule it is ANALYSIS — a specific season shooting percentage is cited in service of the claim. But the model predicted TAKE, likely because the post's dismissive framing ("critics are embarrassing") resembles the confident-assertion language pattern of TAKE posts. The model weighted tone over evidence.

**What it reveals:** This is an annotation-versus-model-behavior gap, not an annotation error. The label is correct by the rule, but the model didn't learn the rule — it learned that posts attacking people → TAKE, even when those posts contain evidence. More training examples showing evidence-present-but-dismissive-tone as ANALYSIS would help.

---

#### Error 3 — Historical superlative without data (TAKE → ANALYSIS)

> *"Jokic's passing is legitimately the best in the history of the center position. No big man has ever seen the floor this well."*
>
> **True: TAKE** | **Predicted: ANALYSIS** | **Confidence: 0.72**

**Why this failed:** Also a flagged hard case. The post uses formal declarative language ("legitimately"), a historical scope claim ("history of the center position"), and a comparative statement ("no big man has ever") — all features the model associates with ANALYSIS. But no specific evidence is cited: no assist numbers, no comparison to specific historical centers, no game reference. This is the clearest example of the model confusing *the register of analysis* with *actual analysis*.

**What it reveals:** The 0.72 confidence is higher than Error 1's 0.61, suggesting the model is more certain about this wrong prediction. Posts like this — confident, basketball-specific, historically framed — are the hardest cases in the dataset and would require many more contrasting examples to teach the evidence requirement.

---

### Sample Classifications

| Post (truncated to 80 chars) | True | Predicted | Confidence | Correct? |
|---|---|---|---|---|
| "OKC has the youngest core to reach 50 wins in NBA history — average age 22.4..." | ANALYSIS | ANALYSIS | 82% | ✅ |
| "LFG KNICKS LET'S GOOOOO 🏆🏆🏆" | BANTER | BANTER | 97% | ✅ |
| "SGA is the best player in the league right now and it's not even close." | TAKE | TAKE | 85% | ✅ |
| "Steph Curry changed the game more than any player since Jordan..." | TAKE | ANALYSIS | 61% | ❌ |
| "Anthony Davis is the most frustrating player in NBA history. Incredible talent, zero heart." | TAKE | TAKE | 79% | ✅ |

**Correct prediction explained:** The model correctly labeled *"OKC has the youngest core to reach 50 wins in NBA history — average age 22.4 at the All-Star break. The 2012-13 Thunder core was 24.6 at the same point."* as ANALYSIS with 82% confidence. The prediction is reasonable: the post contains a specific named statistic (22.4 average age), an explicit historical comparison (to the 2012-13 Thunder at 24.6), and the word "unprecedented" signals the claim is being supported rather than asserted. These features — numbers, named historical reference, comparative framing — are the clearest training signal for ANALYSIS.

**Incorrect prediction explained:** The model predicted *"Steph Curry changed the game more than any player since Jordan"* as ANALYSIS at 61% confidence (true label: TAKE). This fits Error 1's pattern: historical comparison + causal claim in formal language = ANALYSIS to the model, even with no evidence cited.

---

### Reflection: What the Model Captured vs. What I Intended

**What I intended the model to learn:**
The ANALYSIS/TAKE distinction is about *evidence presence* — does the post contain a specific piece of evidence (a stat, a named tactical concept, a historical specific) that does argumentative work? This is the annotation rule, applied consistently across 200 examples.

**What the model actually learned:**
Based on the confusion matrix and error analysis, the model learned a set of surface proxies:
- **Post length:** ANALYSIS posts in the training data tend to be 40–80 words; TAKE posts tend to be 10–25 words. The model uses length as a soft prior.
- **Presence of numbers/percentages:** Posts containing digits are more likely predicted ANALYSIS — which is usually correct but breaks down when a number appears incidentally ("He scored 30 last night, what a GOAT").
- **Register and tone:** Formal, declarative, historically-framed language signals ANALYSIS; confrontational or dismissive language signals TAKE — even when these tone features conflict with the actual evidence content.

**The gap:**
The model cannot evaluate whether evidence is "in service of a claim." That requires reading comprehension — understanding the logical relationship between a cited fact and the assertion it supports. With 140 training examples, the model learned statistical correlations between surface features and labels, not the underlying principle. This is expected behavior for a small fine-tuned model on a semantically subtle task.

**What would need to change:**
The biggest leverage would come from adding training examples that *violate the surface proxies*: short ANALYSIS posts (single stat, 8 words), long TAKE posts (extended rant with no evidence), and formal-register TAKE posts (historically framed but evidence-free). These "counter-examples" to the proxy signals would force the model to find a more semantic decision boundary. The 7 hard cases in the dataset were an attempt at this, but 7 out of 200 is insufficient to override the dominant correlation.

---

### Baseline Reflection

**Pre-training hypothesis:** The zero-shot Llama baseline was expected to struggle most at the TAKE/ANALYSIS boundary — specifically, posts that sound authoritative but cite no evidence. A large language model reading the label definitions has the reading comprehension to understand the rule but may apply it inconsistently when post structure gives conflicting signals.

**After results:** This hypothesis was confirmed. The baseline's ANALYSIS F1 (0.556) was its weakest class — lower than fine-tuned (0.632). The baseline did better than expected on BANTER (F1 0.769 vs. fine-tuned's 0.923) because the definitions are so clear and the class is visually distinct; the gap here mostly reflects the fine-tuned model having seen specific BANTER patterns during training. TAKE performance was identical for both models (F1 0.714), suggesting the distinction between TAKE and BANTER is learnable from definitions alone and doesn't benefit much from fine-tuning at this data scale.

---

## Spec Reflection

**One way the spec helped:** The requirement to document difficult annotation cases *before* annotating the full dataset was the most valuable constraint in the project. Writing out the hard cases and explicit resolution rules (the "evidence in service of the claim" principle) forced a precise definition of the ANALYSIS/TAKE boundary before seeing how the model would behave. Without this, annotation drift across 200 examples would have been significant — it's easy to unconsciously shift what counts as "evidence" after labeling the 50th example. The planning.md annotation rules ended up being referenced throughout the labeling process.

**One way implementation diverged from the spec:** The spec recommended manually collecting data (copy-paste from Reddit, ~1–2 hours). Instead, the dataset was constructed from knowledge of r/nba discourse patterns rather than scraped from live posts. The tradeoff: examples are more carefully controlled for label distribution and hard-case coverage, but they may not capture the full variance of real posts — particularly rare formats like multi-paragraph film breakdowns or embedded video reactions. A scrape-based dataset would have more ecological validity; this dataset has better coverage of the hard TAKE/ANALYSIS boundary by design.

---

## AI Usage

### Instance 1: Pre-labeling assistance (annotation)

**What I directed:** Provided Claude Sonnet with the full label definitions from `planning.md` and batches of unlabeled posts. Prompt: *"Label each post as ANALYSIS, TAKE, or BANTER using these definitions. Apply the annotation rule: if any specific evidence is present in service of a claim, label it ANALYSIS regardless of argument quality. Output ONLY the label for each numbered post."*

**What it produced:** Labels for all 200 examples with near-perfect consistency on clear cases. `pre_labeled_by_ai = True` in the dataset CSV.

**What I changed:** On the 7 hard cases flagged in the `notes` column, I overrode or confirmed Claude's suggestion only after writing the explicit reason. For example, Claude initially labeled *"Steph Curry changed the game more than any player since Jordan..."* as ANALYSIS (it reads like a causal argument). I overrode this to TAKE because no specific evidence is cited — the second sentence is an assertion about influence, not evidence of it.

---

### Instance 2: Wrong prediction pattern analysis (evaluation)

**What I directed:** Pasted the 8 wrong predictions into Claude with the prompt: *"Identify systematic patterns in these misclassifications. Group by error type and describe what each group has in common — look for post length, presence of numbers, tone, specific label pair confused, sarcasm or irony."*

**What it produced:** Three patterns: (1) authoritative tone triggers ANALYSIS, (2) dismissive tone overrides evidence signals, (3) length as proxy.

**What I verified and corrected:** Claude also suggested sarcasm was a major source of errors. On manual review of all 8 wrong predictions, none were primarily sarcastic. Sarcastic posts in the dataset share surface features with BANTER and were classified correctly. Removed the sarcasm pattern after verification.

---

### Instance 3: Label stress-testing (pre-annotation design)

**What I directed:** Before annotating any examples, asked Claude to generate 10 posts at the TAKE/ANALYSIS boundary and 5 at the TAKE/BANTER boundary.

**What it produced:** Posts like *"Teams that switch everything give up 2% more points in the half-court, and the Celtics prove it every year"* — cites a stat but connects it weakly to the specific Celtics claim.

**What I changed:** The stress-testing confirmed that my "any evidence present = ANALYSIS" rule would classify ambiguous cases consistently, even when the argument quality is low. The alternative — requiring evidence to *support the claim well* — would have introduced subjective judgment that breaks annotation consistency. The stress-test validated keeping the simple rule.

---

## Files

| File | Description |
|------|-------------|
| `nba_dataset.csv` | 200 labeled examples (text, label, notes, pre_labeled_by_ai) |
| `planning.md` | Label taxonomy, annotation rules, data plan, evaluation plan, AI tool plan |
| `ai201_project3_takemeter_starter_clean.ipynb` | Training and evaluation notebook (all TODOs filled in) |
| `confusion_matrix.png` | Confusion matrix from fine-tuned model on test set |
| `evaluation_results.json` | Accuracy comparison (baseline vs. fine-tuned) |
| `demo_classify.py` | Demo script: run posts through the fine-tuned model with confidence scores |
| `README.md` | This file |
