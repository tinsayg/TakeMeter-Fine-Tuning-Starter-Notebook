# TakeMeter Planning Document
## r/nba Discourse Quality Classifier

---

## Community

**Chosen community:** r/nba (17M+ members, one of the largest sports communities on Reddit)

**Why this community:** r/nba is a natural fit for discourse quality classification because it produces enormous volume across a genuine quality spectrum — the same event (a single game) generates everything from tactical film-room breakdowns to pure emoji reactions within the same thread. The community has strong implicit norms: regulars can immediately recognize the difference between someone who actually watched the film and someone who had a knee-jerk reaction to a box score. What makes the discourse varied enough to be interesting is that the TAKE/ANALYSIS boundary is genuinely blurry — a post that cites one well-chosen stat looks superficially like analysis but may be cherry-picking, and a post with no stats but sharp tactical observation is more analytical than most stat-heavy comments. This makes it a harder and more interesting classification problem than simple sentiment or topic detection.

**Community description (for README):** r/nba is a 17-million-member Reddit community that serves as the central hub for NBA fan discourse, covering games, players, trades, history, and league culture. Every game produces hundreds of posts spanning from evidence-backed tactical breakdowns to pure emotional reactions, with everything in between. The meaningful quality distinction in this space is not positive vs. negative, or correct vs. incorrect — it is whether a post makes an argument you can actually engage with or simply asserts a position as if it were obvious.

---

## Label Taxonomy

### Label 1: ANALYSIS

**Definition:** The post makes a basketball claim and supports it with at least one piece of specific evidence — a statistic, tactical observation, historical comparison, or film-based reasoning. The argument has substance that requires engagement to rebut; you cannot dismiss it by saying "that's just your opinion."

**Annotation rule:** If the post presents at least one specific supporting element (a cited stat, a named tactical concept, a historical parallel with specifics), label it ANALYSIS — even if the conclusion is debatable or the reasoning is thin. Evidence does not have to be comprehensive or correct, just present and in service of the claim.

**Example 1 (clear ANALYSIS):**
> "Wembanyama isn't just blocking shots — opponents shoot 18% below league average on possessions where he's the last defender before the rim. That's not shot-blocking, that's defensive gravity. No other big in the league changes opponent behavior before contact like this."

*Why this is ANALYSIS:* Cites a specific comparative stat, explains the mechanism (deterrence vs. blocking), and makes a claim that requires engaging with the evidence to rebut.

**Example 2 (clear ANALYSIS):**
> "The reason the Celtics' 3-point defense works isn't athleticism — it's scheme. They switch almost everything on the perimeter and use their length to contest at the arc rather than fighting over screens. Opponents actually attempt more threes against them but make fewer. That's system design, not luck."

*Why this is ANALYSIS:* Names a specific defensive concept (switching scheme vs. fighting over screens), supports the claim with a directional stat comparison, and offers a causal explanation.

---

### Label 2: TAKE

**Definition:** The post expresses a clear basketball opinion or judgment — about a player, team, game, or the league — without meaningful supporting evidence. The claim is stated as assertion rather than argument, as if the conclusion should be obvious to any reasonable person.

**Annotation rule:** If the post makes a basketball judgment but provides no specific evidence (no cited stats, no tactical reasoning, no historical specifics), label it TAKE. This includes "hot takes," player rankings opinions, narrative-driven assessments ("he's soft," "not a winner"), and reactionary game judgments based on impressions rather than data.

**Example 1 (clear TAKE):**
> "SGA is the MVP this year and it's not even debatable. Anyone saying otherwise is coping."

*Why this is TAKE:* Makes a strong basketball judgment with zero supporting reasoning. "Not even debatable" signals the speaker believes the conclusion is self-evident — the hallmark of an assertion, not an argument.

**Example 2 (clear TAKE):**
> "The NBA is too soft now. Players in the 90s played through contact and didn't flop their way to the line every possession. The product has genuinely gotten worse."

*Why this is TAKE:* Expresses a judgment about league quality through era comparison, but offers no stats, no specific examples, no evidence — just assertion backed by nostalgia.

---

### Label 3: BANTER

**Definition:** The post is primarily emotional, social, or reactive with no substantive basketball claim. It exists to express a feeling, participate in community culture, or entertain — not to contribute to basketball discourse. Includes celebration, trash talk, memes, jokes, and content-free reactions.

**Annotation rule:** Ask — "does this post make a basketball judgment that could be agreed or disagreed with on the merits?" If no, it is BANTER. A post can contain basketball references and still be BANTER if its purpose is social/emotional rather than argumentative. Very short posts (under ~8 words), emoji-only posts, and responses to other comments that don't stand alone as positions are almost always BANTER.

**Example 1 (clear BANTER):**
> "LFG KNICKS ARE BACK 🏆🏆🏆"

*Why this is BANTER:* Pure celebration. The "they're back" could theoretically be a judgment, but in context it's emotional expression, not a claim about team quality that could be engaged with.

**Example 2 (clear BANTER):**
> "Lmaooo this league is so cooked 💀💀"

*Why this is BANTER:* Expresses amused frustration but makes no basketball claim. "Cooked" is too vague to constitute a judgment. This post could appear after any play — it has no specific content.

---

## Hard Edge Cases

### Primary hard boundary: TAKE vs. ANALYSIS

**The ambiguous case:**
> "Jaylen Brown shot 42% from three last year. The 'he can't shoot' critics are embarrassing."

This cites a real stat, which triggers the ANALYSIS rule. But it uses one number to make a sweeping character judgment and attacks the critics rather than building an argument. Is this analysis with one data point, or a take dressed up in a stat?

**How I will handle it:** ANALYSIS. My annotation rule is clear — *any* specific evidence in service of the claim qualifies, even if the reasoning is thin or the conclusion overclaims. This prevents me from using my personal judgment about whether the argument is *good* analysis, which would make the labeling inconsistent. The model will learn the underlying signal (evidence-present vs. evidence-absent) rather than my opinion about argument quality. Posts where a stat is cited purely as decoration ("he scored 30 last night, what a GOAT") with no connection to the claim will be TAKE, because the stat is not in service of the argument.

### Secondary hard boundary: TAKE vs. BANTER

**The ambiguous case:**
> "The Knicks are so back 😤"

This expresses a judgment ("they're back" = this team is now good) but does so in an emotionally coded, meme-inflected way that reads more like celebration than opinion.

**How I will handle it:** BANTER. My tie-breaker for TAKE/BANTER is: *could this post appear in a debate without feeling out of place?* "The Knicks are so back" is not a position you argue — it is a mood. TAKE requires that the post is expressing a judgment the author believes, not just participating in communal hype. Short, emoji-heavy, game-thread-style posts lean BANTER even when they contain apparent judgments.

### Expected annotation difficulty

During the 200-example annotation, I expect ~15–20% of posts will require genuine deliberation before labeling. I will flag these in a `hard_case` column (boolean) and record a note explaining the decision. These flagged examples will be priority material for error analysis.

---

## Data Collection Plan

**Source:** Reddit API via PRAW (Python Reddit API Wrapper), pulling from r/nba.

**What I will collect:**
- **Game threads** (during/after specific games): high density of BANTER and reactive TAKES
- **Discussion/opinion posts** (flair: "Discussion", "Highlight", "News + Reaction"): high density of TAKES
- **Film room and stat posts** (flair: "Analysis", "Stats"): high density of ANALYSIS
- **Top comments from high-upvote posts** (cross-section of all types)

**Target distribution:**

| Label    | Target count | Rationale |
|----------|-------------|-----------|
| ANALYSIS | 65          | Hardest to find; must actively seek analytical posts |
| TAKE     | 90          | Most common type; easy to oversample, so cap at 90 |
| BANTER   | 45          | Easy to identify; keep minority to avoid model over-indexing |
| **Total**| **200**     | |

**Train/val/test split:** 140 / 30 / 30 (70% / 15% / 15%), stratified by label.

**If a label is underrepresented after 200 examples:**
- If ANALYSIS < 50: specifically target posts with "film room" or "stat" in the flair, or search for posts containing "per 36", "TS%", "RAPTOR", or similar analytical vocabulary.
- If BANTER > 40% of sample: stop collecting from game threads; shift to discussion posts and top-level comments on opinion threads.
- If TAKE < 60: pull comments from GOAT debate threads, "unpopular opinion" posts, and pre/post-game prediction threads.

**Potential bias to monitor:** Game threads during a playoff run will over-represent the winning team's fans, skewing BANTER toward positive rather than negative affect. Collect game threads from multiple matchups and team communities.

---

## Evaluation Metrics

**Primary metric: Macro F1**
Macro F1 averages F1 across all three classes without weighting by class size. This is the right primary metric because it exposes failure on minority classes — if the model learns to predict TAKE for everything (the largest class), macro F1 will penalize the resulting 0.0 F1 on ANALYSIS and BANTER even if accuracy is decent.

**Secondary metric: Per-class precision and recall**
Reported separately for each label. These reveal the *type* of error:
- High recall, low precision on ANALYSIS = model is over-generous in calling things analysis (labeling TAKES as ANALYSIS)
- High precision, low recall on ANALYSIS = model is too conservative, missing real analytical posts
For a community-facing tool, precision matters more for ANALYSIS (we don't want to falsely highlight low-quality content), while recall matters more for BANTER (we don't want to surface banter in a "best of" feed).

**Supporting metric: Confusion matrix**
A 3×3 matrix showing which pairs of labels the model confuses most. I predict the TAKE/ANALYSIS cell will be the largest off-diagonal entry, which would confirm the label boundary is where the model struggles — validating that our hard edge case analysis was correct.

**Why accuracy alone is insufficient:**
If TAKE is 45% of the test set, a model that predicts TAKE for every single example achieves 45% accuracy while having learned nothing. Macro F1 catches this because ANALYSIS and BANTER F1 would both be 0.0. Accuracy is reported as a sanity check only.

---

## Definition of Success

**Minimum acceptable (model has learned something real):**
- Macro F1 ≥ 0.65 across all three classes
- ANALYSIS F1 ≥ 0.60 specifically (the hardest class; anything below this means the model cannot reliably distinguish evidence-backed posts from assertions)
- Fine-tuned model outperforms zero-shot Llama baseline on macro F1 by at least 5 percentage points

**Good enough for deployment (classifier is genuinely useful):**
- Macro F1 ≥ 0.75
- ANALYSIS precision ≥ 0.70 (when the model flags something as analytical, it's right at least 70% of the time — good enough to surface in a "quality posts" digest without embarrassing misses)
- Confusion between ANALYSIS and BANTER is rare (< 10% of ANALYSIS errors) — these two classes are sufficiently different that systematic confusion between them would indicate a broken model

**How I will objectively determine whether I hit these criteria:**
All thresholds are evaluated on the held-out test set (30 examples), computed in code, and reported as exact numbers. There is no subjective interpretation needed: `macro_f1 >= 0.75` is either true or false. If the fine-tuned model hits 0.74, I will report it as "did not meet the deployment threshold" — not adjust the threshold post-hoc.

---

## AI Tool Plan

### 1. Label Stress-Testing (Before Annotation)

**What I will do:** Give Claude the three label definitions above and ask it to generate 10 posts that sit at the TAKE/ANALYSIS boundary and 5 posts that sit at the TAKE/BANTER boundary. Specifically: "Generate posts that a reasonable person might label either way — not obvious examples, but genuinely ambiguous ones."

**What I am looking for:** If I cannot assign a single label to more than 3 of the 15 generated posts, the definitions are not tight enough and I will revise them before annotating 200 real examples.

**What I will do with the results:** For each post I can't classify, I will identify *why* it is ambiguous and write an explicit rule to resolve it. Those rules go into the "Annotation rule" section of each label definition above. This is already reflected in the TAKE/ANALYSIS rule about "evidence in service of the claim."

### 2. Annotation Assistance

**Decision:** I will use Claude to pre-label batches of 50 posts at a time, with the full label definitions provided in the prompt, before reviewing each label myself.

**Tool:** Claude Sonnet (claude-sonnet-4-6), with the same prompt used for each batch to ensure consistency.

**Tracking:** The dataset CSV will include a `pre_labeled_by_ai` column (boolean) and a `ai_suggested_label` column. My final decision always goes in the `label` column. Any case where my label differs from the AI suggestion will be flagged in `human_override` = True.

**Purpose:** Pre-labeling speeds up annotation and surfaces disagreements that force me to confront the hardest cases explicitly, rather than letting me glide past them.

### 3. Failure Analysis

**What I will do:** After generating test set predictions from both the fine-tuned model and the Llama zero-shot baseline, I will compile all wrong predictions into a list formatted as: `[true_label] → [predicted_label]: "[post text]"` and paste it into Claude with the prompt: "Identify systematic patterns in these misclassifications. Group them by error type and describe what each group has in common."

**What I will look for:**
- Do errors cluster around posts with a single stat cited? (→ evidence quantity boundary issue)
- Do errors cluster around short posts? (→ TAKE/BANTER boundary issue)
- Does the fine-tuned model make the same errors as the baseline, or different ones?

**How I will verify:** For any pattern Claude identifies, I will manually pull the relevant examples and confirm the pattern holds for at least 60% of examples in that cluster before including it in my evaluation report. I will not report a pattern that Claude identified but I cannot verify by hand.

---

## Notes on Scope

- Posts are comments or top-level text posts. Image-only posts and video highlights with no text are excluded.
- Non-English posts are excluded.
- Posts under 5 words are labeled BANTER by default (they cannot contain enough information to constitute ANALYSIS or TAKE).
- Moderator posts and AutoModerator messages are excluded.
