# TakeMeter Planning Doc
## Project: Rainbow Six Siege Discourse Classifier

---

## Community

I chose r/Rainbow6, the main subreddit for Rainbow Six Siege with ~400K weekly
visitors. This community is a strong fit for a classification task because the
discourse is highly varied — you see everything from detailed tactical breakdowns
to emotional rants about Ubisoft to simple questions from new players. The
distinctions between post types are real and meaningful to people in the
community; regulars genuinely react differently to a strategy post vs. a hot take
vs. a complaint thread.

---

## Labels

I defined 4 labels:

**strategy**
A post that discusses mechanics, operator picks, map angles, or tactics with
specific reasoning that could be applied in-game.
- Example 1: "Some Hibana angles I found on Casino basement — these also work
  with any DMR on defense. Azami is especially strong on cash extension."
- Example 2: "If you put a hole next to Mira's window you can wallbang anyone
  who peeks the mirror without exposing yourself."

**critique**
A post expressing criticism directed at Ubisoft, game design decisions, or
balance choices — dissatisfaction with how the game is run.
- Example 1: "Does anyone at Ubisoft know what a bathroom looks like?" (mockery
  of map design)
- Example 2: "This is a huge downgrade." (responding to a UI or operator change)

**hot_take**
A bold opinion about the meta, operators, or community stated confidently without
supporting reasoning or evidence.
- Example 1: "What POSSIBLE reason could there be to ban Wamai off rip?"
- Example 2: "Sledge is criminally underrated and anyone who disagrees is wrong."

**question**
A post asking for help, advice, or information from the community.
- Example 1: "When should I buy premium? I'm 30 hours in and still learning."
- Example 2: "Can someone explain why people put holes next to Mira's windows?"

---

## Hard Edge Cases

**question vs. critique**
Some posts are phrased as questions but are really venting frustration.
- Ambiguous example: "Am I just bad or is cav not as good as she used to be?"
- Decision rule: If the post is primarily seeking an answer or validation, label
  it `question`. If it's primarily expressing frustration at the game even in
  question form, label it `critique`. A question mark alone doesn't make it a
  `question`.

**hot_take vs. critique**
Both can express strong negative opinions, but the target differs.
- Ambiguous example: "Dok is OP and Ubisoft refuses to fix it."
- Decision rule: If frustration is directed at Ubisoft's decisions or game
  design, it's `critique`. If it's a bold claim about the meta or players with
  no blame directed at Ubisoft, it's `hot_take`.

**strategy vs. hot_take**
Some posts cite mechanics but are still mostly asserting rather than reasoning.
- Ambiguous example: "Jäger is overpowered — his ADS clears nades too fast."
- Decision rule: If the post gives specific, applicable mechanical reasoning
  (angles, timings, operator interactions) that could help someone play better,
  it's `strategy`. If it's a strong opinion with only surface-level justification,
  it's `hot_take`.

---

## Data Collection Plan

- Source: r/Rainbow6 public posts (manually copied from the subreddit feed and
  comment sections)
- Target: 200+ examples, aiming for ~50 per label
- If a label is underrepresented after 150 examples, I will specifically seek
  out posts of that type (e.g. browse r/SiegeAcademy for more strategy posts)
- I will save all examples in a single CSV with columns: `text`, `label`, `notes`
- The notebook handles the train/val/test split automatically (70/15/15)

---

## Evaluation Metrics

I will report:
- **Overall accuracy** for both the fine-tuned model and the Groq baseline
- **Per-class F1** for each of the 4 labels — accuracy alone isn't enough
  because a model could get high accuracy by over-predicting the majority class
- **Confusion matrix** to identify which label pairs are being confused and in
  which direction

F1 is the right per-class metric here because the classes may not be perfectly
balanced and I care equally about precision and recall for each label.

---

## Definition of Success

I will consider the classifier genuinely useful if:
- Overall accuracy exceeds 75% on the test set
- No single label has an F1 below 0.60
- The fine-tuned model meaningfully outperforms the Groq zero-shot baseline

A model that hits these thresholds could realistically be used as a post-flair
suggester or discourse quality filter in a community tool.

---

## AI Tool Plan

**Label stress-testing:** I will give Claude my label definitions and edge case
rules and ask it to generate 10 boundary posts — posts that could plausibly
belong to two labels. If I can't classify them cleanly, I'll tighten the
definitions before annotating.

**Annotation assistance:** I may use an LLM to pre-label a batch of examples
using my definitions, then review and correct every label myself. If I do this,
I will track which examples were pre-labeled and disclose it in the AI usage
section of my README.

**Failure analysis:** After training, I will paste my wrong predictions into
Claude and ask it to identify patterns before writing my evaluation. I will
verify any patterns it identifies by re-reading the examples myself.
