# TakeMeter — Rainbow Six Siege Discourse Classifier

A fine-tuned text classifier that evaluates discourse quality in the Rainbow Six Siege subreddit (r/Rainbow6 and r/SiegeAcademy). Built as part of AI201 Project 3.

---

## Community

I chose r/Rainbow6 and r/SiegeAcademy, the primary Reddit communities for Rainbow Six Siege with ~400K and ~216K weekly visitors respectively. This community is a strong fit for a classification task because the discourse is highly varied — the same day you'll see detailed tactical breakdowns, emotional rants about Ubisoft, new player questions, and bold meta opinions. These distinctions are real and meaningful to people in the community; regulars genuinely react differently to a strategy post versus a complaint thread versus a hot take.

---

## Label Taxonomy

I defined 4 labels:

**strategy** — A post that discusses mechanics, operator picks, map angles, or tactics with specific reasoning that could be applied in-game.
- Example 1: "Some Hibana angles I found on Casino basement — Azami is especially strong on cash extension and makes entering from servers basically impossible."
- Example 2: "If you're solo queuing I recommend Maverick for attack and Doc for defense — both give you independence from teammates."

**critique** — A post expressing criticism directed at Ubisoft, game design decisions, or balance choices.
- Example 1: "Things to fix in Siege right now — fix low FPS, fix match replay crashes, bring back the Marketplace."
- Example 2: "R6 cheaters — why does it feel like there are more cheaters this season? Ubisoft's anti-cheat got worse."

**hot_take** — A bold opinion about the meta, operators, or community stated confidently without supporting reasoning or evidence.
- Example 1: "If you ban Ace on 3F Kafe you need a brain transplant."
- Example 2: "Ranked 3.0 is not broken, you just aren't as good as you thought."

**question** — A post asking for help, advice, or information from the community.
- Example 1: "Can someone explain why people put holes next to Mira's windows?"
- Example 2: "When should I buy premium? I'm 30 hours in and still learning."

---

## Data Collection

**Source:** Public posts and titles from r/Rainbow6 and r/SiegeAcademy, collected manually by browsing the subreddit feed across multiple sessions.

**What was excluded:** Image posts, video/gameplay clips, meme/fluff posts, and any post whose title alone was too short or vague to label without seeing the image. These were excluded because the classifier operates on text only.

**Labeling process:** Each post was labeled using the definitions above. Posts that gave genuine pause were noted in the CSV notes column.

**Label distribution (full dataset):**
| Label | Count |
|---|---|
| hot_take | ~53 |
| strategy | ~51 |
| question | ~80 |
| critique | ~33 |

After trimming question posts to reduce imbalance, the final split was:
- Train: 139 examples
- Validation: 30 examples
- Test: 30 examples

**Three difficult-to-label examples:**

1. *"Am I just bad or is cav not as good as she used to be?"* — Could be `question` (seeking validation) or `critique` (frustration at game changes). Decision: `question` because the post is primarily seeking an answer, not venting at Ubisoft.

2. *"The state of unranked for a new player is unbearable"* — Could be `critique` (directed at Ubisoft matchmaking) or `hot_take` (bold opinion). Decision: `critique` because the frustration has a clear target — Ubisoft's matchmaking design.

3. *"Is Ace really that much better than Hibana?"* — Could be `question` or `strategy`. Decision: `strategy` because it's asking about a tactical operator comparison that requires game knowledge to answer, not just community information.

---

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` from HuggingFace

**Training setup:** Fine-tuned on Google Colab with a T4 GPU using the HuggingFace `transformers` and `datasets` libraries.

**Hyperparameter decisions:**
- Initial run used default settings (3 epochs, lr=2e-5, batch size=16) — model failed to learn, stuck at 30% validation accuracy across all epochs.
- Adjusted to 5 epochs, lr=3e-5, batch size=8. The smaller batch size helped with the small dataset size, and extra epochs gave the model more time to learn. Validation accuracy improved to 63% by epoch 5 with a consistently decreasing training loss (1.41 → 0.88).

---

## Baseline

**Model:** Groq `llama-3.3-70b-versatile`, zero-shot (no task-specific training)

**Prompt used:**
```
You are classifying posts from the Rainbow Six Siege subreddit (r/Rainbow6 and r/SiegeAcademy).
Assign each post to exactly one of the following categories.

strategy: discusses mechanics, operator picks, map angles, or tactics with specific reasoning that could be applied in-game.
critique: criticism directed at Ubisoft, game design decisions, or balance choices.
hot_take: a bold opinion about the meta, operators, or community stated confidently without supporting reasoning.
question: asking for help, advice, or information from the community.

Respond with ONLY the label name. Do not explain your reasoning.
```

All 30 test examples returned parseable responses.

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
|---|---|
| Groq baseline (zero-shot) | 73.3% |
| Fine-tuned DistilBERT | 63.3% |

### Per-Class Metrics

**Baseline (Groq llama-3.3-70b-versatile):**
| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| strategy | 1.00 | 0.62 | 0.77 | 8 |
| critique | 0.80 | 0.80 | 0.80 | 5 |
| hot_take | 1.00 | 0.56 | 0.71 | 9 |
| question | 0.53 | 1.00 | 0.70 | 8 |

**Fine-tuned DistilBERT:**
| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| strategy | 0.75 | 0.75 | 0.75 | 8 |
| critique | 0.00 | 0.00 | 0.00 | 5 |
| hot_take | 0.62 | 0.56 | 0.59 | 9 |
| question | 0.57 | 1.00 | 0.73 | 8 |

### Confusion Matrix (Fine-Tuned Model)

| | Predicted: strategy | Predicted: critique | Predicted: hot_take | Predicted: question |
|---|---|---|---|---|
| **True: strategy** | 6 | 0 | 1 | 1 |
| **True: critique** | 0 | 0 | 2 | 3 |
| **True: hot_take** | 2 | 0 | 5 | 2 |
| **True: question** | 0 | 0 | 0 | 8 |

The model never predicted `critique` once on the test set. All 5 critique examples were misclassified — 3 as `question` and 2 as `hot_take`.

### Wrong Prediction Analysis

**#1 — "Why did they worsen the graphics and can we bring it back"**
- True: `critique` | Predicted: `question` (confidence: 0.40)
- The post is phrased as a question ("can we bring it back") but its target is Ubisoft's decision to change graphics — making it critique. The model latched onto the question syntax rather than the underlying intent. This is a labeling boundary issue: critique posts in R6 are often phrased rhetorically as questions. A tighter decision rule distinguishing rhetorical questions from genuine ones would help.

**#2 — "Is Ace really that much better than Hibana?"**
- True: `strategy` | Predicted: `question` (confidence: 0.44)
- This is a genuine edge case. The post looks like a question syntactically, but the intent is to discuss operator tradeoffs — a tactical comparison. The model correctly identified the question structure but missed the strategic content. More strategy examples phrased as questions would help train this boundary.

**#3 — "If you ban Ace on 3F Kafe you need a brain transplant"**
- True: `hot_take` | Predicted: `strategy` (confidence: 0.36)
- The post contains a specific map name (3F Kafe) and operator (Ace) which are strong signals the model associates with strategy. But the post is asserting an opinion rather than reasoning through tactics. The model overfit to map/operator name mentions as strategy indicators, missing the assertive framing that defines a hot_take.

### Sample Classifications

| Post | True Label | Predicted | Confidence |
|---|---|---|---|
| "Some Hibana angles I found on Casino basement — Azami is especially strong on cash extension" | strategy | strategy | 0.81 |
| "Can someone explain why people put holes next to Mira's windows?" | question | question | 0.79 |
| "Ranked 3.0 is not broken, you just aren't as good as you thought." | hot_take | hot_take | 0.67 |
| "Does anyone at Ubisoft know what a bathroom looks like?" | critique | question | 0.39 |
| "Hot take: kills matter more than you think" | hot_take | strategy | 0.38 |

The strategy prediction for the Hibana angles post is reasonable — the post explicitly names a map, an operator, and a specific utility placement that a player could apply in their next game. This is exactly the kind of actionable tactical content the strategy label was designed for.

---

## Reflection: What the Model Learned vs. What I Intended

I intended the model to learn the *intent* behind each post — whether someone was sharing tactics, criticizing Ubisoft, asserting an opinion, or seeking help. What the model actually learned was closer to surface-level syntactic patterns.

For `question`, it learned to recognize question marks and interrogative phrasing — which worked well (recall 1.00) but caused it to over-predict the label, pulling in critique posts phrased as rhetorical questions. For `strategy`, it learned to recognize operator and map name mentions, which fired correctly on genuine strategy posts but also misfired on hot_takes that happened to name specific operators or maps. For `critique`, it learned almost nothing — likely because 23 training examples wasn't enough, and because critique posts in R6 look superficially similar to both hot_takes (strong opinions) and questions (rhetorical phrasing).

The fundamental gap is that my labels are defined by *intent and target* (who or what is the post directed at, and what is it trying to do) but the model can only see *surface text*. A post criticizing Ubisoft and a post asking a question can look nearly identical on the surface if both are phrased as questions. More labeled data and more diverse examples of the critique/hot_take boundary would be the most impactful fix.

---

## Spec Reflection

**One way the spec helped:** The requirement to define edge cases and decision rules before annotating was genuinely valuable. Writing down the `question vs. critique` decision rule (a question mark alone doesn't make it a question) forced me to think about the boundary before I hit it 50 times during annotation. Without that, my labeling would have been much more inconsistent.

**One way implementation diverged:** The spec suggested aiming for ~50 examples per label, but my `question` label ended up significantly larger (~80 before trimming) because question posts are by far the most common type on both subreddits. I trimmed aggressively but still ended up with more questions than other labels. In hindsight I should have capped question collection earlier and spent more time hunting for critique posts specifically.

---

## AI Usage

**1. Label taxonomy and edge case design (Claude):** I used Claude throughout Milestone 1 to develop and stress-test my label definitions. Claude proposed the initial 4-label taxonomy, helped me identify the three main edge cases (question vs. critique, hot_take vs. critique, strategy vs. hot_take), and drafted the decision rules I used during annotation. I reviewed and adjusted the rules — for example, the initial hot_take vs. critique rule was sharpened after I found several posts that didn't fit cleanly.

**2. Annotation assistance (Claude):** I pasted pages of r/Rainbow6 and r/SiegeAcademy posts into Claude and asked it to label each usable post with one of my four labels and a brief note explaining the decision. Claude labeled approximately 200 posts across multiple sessions. I reviewed every label Claude assigned before adding it to my CSV, correcting roughly 15–20% of them — most commonly where Claude labeled rhetorical critique posts as `question`, or where it labeled short hot_takes about Ubisoft as `critique`.

**3. Failure pattern analysis (Claude):** After running the fine-tuned model, I pasted the 11 wrong predictions into Claude and asked it to identify common patterns. Claude identified two main patterns: (1) the model was confusing posts with question syntax for the question label regardless of intent, and (2) operator/map name mentions were pulling posts toward the strategy label. I verified both patterns by re-reading the wrong predictions myself — both held up. I also noted a third pattern Claude missed: the model's complete failure on `critique` is largely explained by the low training count (23 examples), not just surface text confusion.
