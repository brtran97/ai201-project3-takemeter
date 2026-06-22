# TakeMeter: Classifying r/nba Discourse Quality

[Demo Video](https://drive.google.com/file/d/17jlt3GWJqTn0ykPtsAXFuNd-WzrX6jNi/view?usp=sharing)

## Community Choice

**Community**: r/nba and r/nbadiscussion on Reddit

r/nba is one of the largest and most active sports communities on Reddit, generating constant text-heavy discourse that ranges from careful statistical analysis to emotional outbursts during game threads. The community itself has an informal vocabulary for discourse quality — users regularly call out "hot takes," praise detailed breakdowns, and mock low-effort reactions. These distinctions are real to participants, making it a natural fit for a classification task. Data is publicly accessible, high-volume, and varied enough across thread types (game threads, post-game discussion, offseason analysis) to produce meaningful label diversity.

## Label Taxonomy

Three labels capturing the spectrum of discourse quality on r/nba:

**analysis** — The post makes a structured argument backed by statistics, historical comparison, or tactical observation. Evidence is specific and verifiable.

Examples:
- "It's the highest percentage of a team's points in a Finals game (47.9%) since MJ in game 6 in 1998"
- A post arguing Fox criticism should be more nuanced, citing career stats (27ppg peak, 51.2 FG%), age comparisons, and playoff history to support a structured argument about player development.

**hot_take** — A bold, confident opinion stated without supporting evidence. The claim might be true, but the post asserts rather than argues.

Examples:
- "what makes it funnier is the reason he got waived. they needed to make room for mac mcclung so he could wear their jersey in the dunk contest. they waived a finals starter for the dunk contest."
- "perhaps we shouldnt be calling teams a dynasty after 1 chip lol"

**reaction** — An immediate emotional response to a specific event. Little to no argument — the post expresses a feeling in the moment.

Examples:
- "Meanwhile Chet is taking 2 shots on a max"
- "From a historical half-time Finals lead to an all-time choke job. This series is on crack."

### Decision Rules

- **hot_take vs. reaction**: If the post makes a claim (even implied) about a player's value, a team's decisions, or the league — hot_take. If it expresses a feeling without a broader position — reaction.
- **hot_take vs. analysis**: If evidence is specific, verifiable, and part of a structured argument — analysis. If stats are cherry-picked or decorative — hot_take.

## Annotated Dataset

### Data Source and Labeling Process

Data was collected from r/nba and r/nbadiscussion by downloading Reddit JSON files from post threads covering the 2025 NBA Finals and Western Conference Finals. Comments were extracted programmatically (top-level comments and first direct replies), filtered to remove bot comments, deleted posts, and comments under 10 words. An LLM (Claude) was used to propose initial labels based on the label definitions and decision rules, and all labels were then manually reviewed and corrected.

### Label Distribution

| Label | Count | Percentage |
|-------|-------|------------|
| hot_take | 116 | 51.1% |
| reaction | 60 | 26.4% |
| analysis | 51 | 22.5% |
| **Total** | **227** | **100%** |

No single label exceeds 70%. Hot_take is the most common because r/nba discourse naturally skews toward short, opinionated assertions. Analysis is the rarest because it requires structured arguments with evidence, which is less common in comment threads.

### Difficult-to-Label Examples

1. **Champagnie/McClung comment** — "what makes it funnier is the reason he got waived. they needed to make room for mac mcclung so he could wear their jersey in the dunk contest. they waived a finals starter for the dunk contest." Could be reaction (commenting on an amusing situation) or hot_take (implying the 76ers made a bad decision). **Decision**: hot_take, because the narrative structure builds toward an implied claim about the team's decision-making.

2. **Chet "2 shots on a max"** — Could be reaction (frustration at his game performance) or hot_take (implying he's not worth his contract). **Decision**: reaction, because "2 shots" describes what just happened in the game and the contract mention is color, not a developed argument about his value.

3. **"dynasty after 1 chip"** — Could be reaction (responding to OKC's loss) or hot_take (making a general claim about how dynasties should be defined). **Decision**: hot_take, because the statement goes beyond the moment to make a broader claim about the league.

## Fine-Tuning Approach

### Base Model and Platform

- **Model**: `distilbert-base-uncased` from HuggingFace
- **Platform**: Google Colab with T4 GPU
- **Training libraries**: `transformers`, `datasets`, `scikit-learn`

### Key Training Decisions

The default hyperparameters (3 epochs, lr 2e-5, batch size 16) caused the model to collapse to always predicting hot_take — the majority class at 51% of training data. After iterating through several configurations, the final training setup was:

- **Epochs**: 5 (increased from 3 to give the model more passes to learn minority classes; higher values like 8-10 showed diminishing returns)
- **Learning rate**: 2e-5 (kept at default — lower rates like 5e-6 led to underfitting)
- **Class weights**: Inverse frequency weighting via a custom `WeightedTrainer` with `CrossEntropyLoss(weight=class_weights)` — this was the single most impactful change, preventing the model from ignoring minority labels. Without class weights the model predicted hot_take for 100% of test examples.

## Baseline Comparison

### Baseline Approach

The zero-shot baseline used Groq's `llama-3.3-70b-versatile` model with the following system prompt:

```
You are classifying posts and comments from r/nba (the NBA subreddit).
Assign each post to exactly one of the following categories.

analysis: The post makes a structured argument backed by statistics, historical
comparison, or tactical observation. Evidence is specific and verifiable.
Example: "It's the highest percentage of a team's points in a Finals game (47.9%)
since MJ in game 6 in 1998"

hot_take: A bold, confident opinion stated without supporting evidence. The claim
might be true, but the post asserts rather than argues.
Example: "perhaps we shouldnt be calling teams a dynasty after 1 chip lol"

reaction: An immediate emotional response to a specific event. Little to no
argument — the post expresses a feeling in the moment.
Example: "From a historical half-time Finals lead to an all-time choke job. This
series is on crack."

Respond with ONLY the label name.
```

Each test example was sent individually with `temperature=0`. All 35 responses were parseable.

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
|-------|----------|
| Zero-shot baseline (Groq) | 48.6% |
| Fine-tuned DistilBERT | 42.9% |

The fine-tuned model has lower overall accuracy than the baseline. However, this comparison is misleading — the baseline achieves its accuracy by over-predicting reaction (89% recall) while almost completely missing hot_take (22% recall). The fine-tuned model distributes predictions more evenly across all three labels.

### Per-Class Metrics

**Baseline (Groq llama-3.3-70b-versatile):**

| Label | Precision | Recall | F1 | Support |
|-------|-----------|--------|----|---------|
| analysis | 0.45 | 0.62 | 0.53 | 8 |
| hot_take | 0.57 | 0.22 | 0.32 | 18 |
| reaction | 0.47 | 0.89 | 0.62 | 9 |

**Fine-tuned DistilBERT:**

| Label | Precision | Recall | F1 | Support |
|-------|-----------|--------|----|---------|
| analysis | 0.43 | 0.75 | 0.55 | 8 |
| hot_take | 0.38 | 0.17 | 0.23 | 18 |
| reaction | 0.46 | 0.67 | 0.55 | 9 |

The fine-tuned model improves on analysis (F1: 0.53 -> 0.55) and nearly matches baseline on reaction (F1: 0.62 -> 0.55), but performs worse on hot_take (F1: 0.32 -> 0.23). Both models struggle most with hot_take.

### Confusion Matrix

| | Predicted: analysis | Predicted: hot_take | Predicted: reaction |
|---|---|---|---|
| **True: analysis** | 6 | 2 | 0 |
| **True: hot_take** | 8 | 3 | 7 |
| **True: reaction** | 0 | 3 | 6 |

The dominant error pattern is hot_take being misclassified: 8 hot_takes are predicted as analysis and 7 as reaction. The model splits hot_take errors almost evenly between the other two labels, suggesting it has learned what analysis and reaction look like but cannot identify the middle category.

### Error Analysis: 3 Wrong Predictions

**1. True: hot_take, Predicted: analysis (confidence: 0.61)**
> "That's great and all but we have no evidence that Fox will complete that 'maturation process'. There's really nothing to suggest he can turn into an elite game managing point guard, that's never been his style. His shooting is streaky bordering on bad and he's had a few bad mental mistakes and turnovers. He has been pretty bad the entire playoffs 16 ppg on bad efficiency with below average defense..."

The model saw statistics (16 ppg, 55 mil) and argumentative structure, so it predicted analysis with its highest confidence among wrong predictions. But the stats here are decorative — cherry-picked to support a pre-formed conclusion rather than used as part of a genuine analytical framework. This is exactly the hot_take vs. analysis boundary: the model keys on the **presence** of numbers rather than whether they're used structurally.

**2. True: hot_take, Predicted: reaction (confidence: 0.38)**
> "Basketball referees might be the dumbest people on the planet"

The model read the emotional intensity ("dumbest people on the planet") and predicted reaction. But this post is making a claim — a position about referees that could be agreed or disagreed with. The model is using **emotional tone** as a proxy for the reaction label, when the actual distinction is whether a claim is being made. Low confidence (0.38) shows the model was uncertain.

**3. True: hot_take, Predicted: reaction (confidence: 0.42)**
> "This will never not be hilarious because what the actual fuck"

Similar pattern to #2. The profanity and exclamatory tone triggered a reaction prediction, but the post implies a judgment about a team's decision (the Champagnie waiver). The model cannot distinguish **emotional framing of a claim** from a pure emotional outburst.

### Sample Classifications

| Post | Predicted Label | Confidence | Correct? |
|------|----------------|------------|----------|
| "Yes and this is a historical problem with read-and-react offenses. It's rather easy to scout eventually..." | analysis | 0.60 | Yes |
| "The biggest issue for San Antonio going forward is his contract. They're paying him to be their second best player..." | analysis | 0.54 | Yes |
| "This is the most insane game I can recall watching in a long time holy crap lol" | reaction | 0.42 | Yes |
| "This swung the game. OKC probably wins without this block" | hot_take | 0.39 | Yes |
| "Basketball referees might be the dumbest people on the planet" | reaction | 0.38 | No (true: hot_take) |

The first prediction (analysis, 0.60 confidence) is reasonable because the post discusses a tactical concept (read-and-react offenses being easy to scout) and draws on specific basketball strategy knowledge with references to how the offense was exploited in the playoffs. The model correctly identified that the post is building a structured argument about team strategy, not just asserting an opinion.

## Reflection: What the Model Learned vs. What Was Intended

The label taxonomy was designed around a conceptual distinction: **is the person making a claim, and if so, are they backing it up?** Analysis = claim + evidence. Hot_take = claim without evidence. Reaction = no claim at all.

The model learned something different. It learned **surface-level signals**: long posts with numbers tend to be analysis, short emotional posts tend to be reactions. This works for the clear cases at both ends of the spectrum. But hot_take — the middle category defined by what it lacks (evidence) rather than what it contains — has no consistent surface signal. A hot take can be long or short, calm or angry, and may even include a stat or two.

The result is that the model effectively learned a two-class distinction (analysis vs. reaction, roughly corresponding to "long and structured" vs. "short and emotional") and treats hot_take as noise to be distributed between them. The confusion matrix confirms this: hot_take errors split almost perfectly between analysis (8) and reaction (7).

This reveals a fundamental limitation of fine-tuning a small model on 200 examples for a task where one label is defined by absence. The model needs to learn "this post has a claim but NOT sufficient evidence" — a conjunction that requires understanding both what's present and what's missing. DistilBERT with 200 examples learns co-occurrence patterns (stats -> analysis, exclamation marks -> reaction) but not the semantic relationship between claims and their support.

## Spec Reflection

**One way the spec helped**: The spec's emphasis on label stress-testing before annotation was valuable. Generating 8 synthetic boundary posts and classifying them revealed that my decision rules were workable before I committed to labeling 200 examples. Without that step, I might have discovered ambiguity in the hot_take vs. reaction boundary only after annotating inconsistently.

**One way implementation diverged**: The spec suggested annotating all examples manually. Instead, I used an LLM (Claude) to propose initial labels for batches of comments extracted from Reddit JSON files, then reviewed and corrected all labels. This was significantly faster for 227 examples but required vigilance — the LLM tended to over-label comments as analysis when they contained any statistics, even decorative ones. I corrected several of these during review. The spec's caution about pre-labeling was warranted: without careful review, the LLM's biases would have propagated into the training data.

## AI Usage

### Instance 1: Label Stress-Testing

Before annotating, I gave Claude my three label definitions and decision rules, then asked it to generate 8 synthetic r/nba posts that sit at the boundary between two labels (4 on the hot_take/reaction boundary, 4 on the hot_take/analysis boundary). I classified each one using my rules. All 8 were classifiable without needing to revise my definitions, which confirmed the taxonomy was workable. I did notice that the hot_take vs. analysis boundary required careful attention to whether stats were used structurally or decoratively — this informed how I labeled real examples later.

### Instance 2: Annotation Assistance (Pre-Labeling)

I used Claude to parse Reddit JSON files and propose labels for extracted comments based on my label definitions. For each batch, the LLM proposed a label and a short rationale. I reviewed every proposed label and corrected several — for example, one comment ("many spurs fans have been dogging on fox all season...") was proposed as reaction but I relabeled it as hot_take because it was making a claim about why fan criticism exists, not just expressing frustration. The LLM's proposed labels were a useful starting point but required genuine review, not just rubber-stamping.

### Instance 3: Failure Analysis

After training, I reviewed the 20 wrong predictions with Claude to identify systematic patterns. The analysis surfaced a key pattern: the model uses emotional tone as a proxy for the reaction label, causing it to mislabel emotionally-framed hot takes as reactions. I verified this by re-reading the misclassified examples — 7 of the 8 hot_takes mislabeled as reaction contained exclamatory or profane language. This pattern would have been harder to spot reviewing errors one at a time.
