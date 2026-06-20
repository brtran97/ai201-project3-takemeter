# TakeMeter Planning Document

## Community Choice

**Community**: r/nba (Reddit)

r/nba is one of the most active sports communities on Reddit, with millions of subscribers generating high-volume, text-heavy discourse that varies enormously in quality. It is a strong fit for a classification task because the community itself already has an informal vocabulary for discourse quality — users regularly call out "hot takes," praise detailed analysis, and mock low-effort reactions. These distinctions are real to the people who participate, not just academic labels imposed from outside. The mix of game threads (real-time emotional responses), post-game analysis, and offseason debate produces a natural range of discourse types that map cleanly onto a small label taxonomy. Data is publicly accessible and easy to collect manually.

## Label Taxonomy

### Labels (3)

**analysis** — The post makes a structured argument backed by statistics, historical comparison, or tactical observation. Evidence is specific and verifiable.

Examples:
- "It's the highest percentage of a team's points in a Finals game (47.9%) since MJ in game 6 in 1998" — Cites a specific, verifiable statistic to contextualize Jalen Brunson's Game 5 performance with a historical comparison.
- A post arguing that criticism of De'Aaron Fox should be more nuanced: "Fox is not the 30+ year old cerebral point guard that people might associate with the veteran label. He's a 28 year old - younger than Jalen Brunson - with declining athleticism..." — Makes a structured argument using age, career stats (27ppg peak, 51.2 FG%), playoff history, and post-game quotes to support the claim that Fox is still maturing as a player rather than a finished product.

**hot_take** — A bold, confident opinion stated without supporting evidence. The claim might be true, but the post asserts rather than argues.

Examples:
- "what makes it funnier is the reason he got waived. they needed to make room for mac mcclung so he could wear their jersey in the dunk contest. they waived a finals starter for the dunk contest." — Implies the 76ers were foolish for waiving Champagnie, who became a key contributor for the Spurs. Bold implied claim delivered as narrative rather than argued with evidence.
- "perhaps we shouldnt be calling teams a dynasty after 1 chip lol" — Makes a general claim about how the league uses the word "dynasty," triggered by OKC's loss to the Spurs. Asserts rather than argues.

**reaction** — An immediate emotional response to a specific event. Little to no argument — the post expresses a feeling in the moment.

Examples:
- "Meanwhile Chet is taking 2 shots on a max" — Venting frustration about Chet Holmgren's performance during a playoff game. The contract mention adds color but the post is not making a developed argument about his value.
- "From a historical half-time Finals lead to an all-time choke job. This series is on crack." — Reacting to the Knicks' comeback from a 29-point deficit. Pure emotional response to a dramatic moment, no argument or claim being made.

## Hard Edge Cases

The hardest boundary is **hot_take vs. reaction**. Many r/nba comments are short, opinionated, and triggered by a specific event — which makes it unclear whether the person is staking a position or just venting.

### Decision Rule: hot_take vs. reaction

If the post **makes a claim** — even an implied one — about a player's value, a team's decisions, or the state of the league, it is a **hot_take**. The key signal is that the person is staking a position that could be agreed or disagreed with.

If the post **expresses a feeling in the moment** without staking a broader position, it is a **reaction**. The key signal is that the post is emotional and situational — it describes how the person feels, not what they believe.

### Decision Rule: hot_take vs. analysis

If the post **backs up its claim with specific, verifiable evidence** (statistics, game film observations, historical comparisons used as part of a structured argument), it is **analysis**. If the evidence is vague, cherry-picked, or decorative — just enough to sound credible but not genuinely reasoning — it is **hot_take**.

### Edge Case Examples

1. **Champagnie/McClung comment** — "what makes it funnier is the reason he got waived..." Could be reaction (commenting on an amusing situation) or hot_take (implying the 76ers made a bad decision). **Decision**: hot_take, because the narrative structure builds toward an implied claim about the team's decision-making.

2. **Chet "2 shots on a max"** — Could be reaction (frustration about his game performance) or hot_take (implying he is not worth his contract). **Decision**: reaction, because "2 shots" describes what just happened in the game and the contract mention is color, not a developed argument.

3. **"dynasty after 1 chip"** — Could be reaction (responding to OKC's loss) or hot_take (making a general claim about how dynasties should be defined). **Decision**: hot_take, because the statement goes beyond the moment to make a broader claim about the league.

## Data Collection Plan

- **Source**: r/nba on Reddit, browsing directly and manually copying posts and comments into a CSV
- **What to collect**: A mix of full posts (title + body) and comments to capture different lengths and contexts
- **Target**: 200+ labeled examples total
- **Target distribution**: ~67 per label (roughly even three-way split), with no single label exceeding 70% of the dataset
- **Expected imbalance**: Analysis posts tend to be longer and rarer on r/nba because they require structured arguments — most discourse is short-form hot takes and reactions. Analysis will likely be the most underrepresented label.
- **If a label is underrepresented**: Actively seek out threads where that discourse type is more common. Game threads and highlight-clip threads for reactions, offseason and trade-deadline threads for analysis, and post-game threads for hot takes.
- **CSV format**: Columns will be `text`, `label`, and `notes` (for difficult labeling decisions)

## Evaluation Metrics

- **Overall accuracy** for both the fine-tuned model and the zero-shot baseline — the most basic measure, but not sufficient on its own
- **Per-class precision, recall, and F1** — these reveal whether the model is actually learning each label or just predicting the majority class. For example, a model could hit 50% accuracy by only ever predicting hot_take and getting lucky, while completely ignoring analysis and reaction.
- **Confusion matrix** — shows which specific label pairs the model confuses and in which direction. A large off-diagonal value at (true=analysis, predicted=hot_take) would mean the model cannot distinguish evidence-backed arguments from bare assertions — the most important boundary in this taxonomy.

Accuracy alone is insufficient because with 3 roughly balanced labels, a trivial classifier that always predicts the most common label would score ~33%. Per-class F1 ensures the model is learning all three distinctions, not just one. The confusion matrix gives actionable information about which boundary to investigate or tighten.

## Definition of Success

A successful classifier meets all three of these thresholds:

1. **Fine-tuned accuracy ≥ 60%** on the held-out test set
2. **Fine-tuned model beats the zero-shot Groq baseline by ≥ 10 percentage points**
3. **No single label has F1 below 0.40** — the model must learn all three distinctions, not collapse two labels into one

These thresholds are set for a subjective 3-class task with only 200 training examples. 60% accuracy is nearly double the random baseline of 33%, and a 10-point improvement over zero-shot indicates that fine-tuning captured something the general model could not. An F1 floor of 0.40 per class ensures the model is not achieving overall accuracy by ignoring the hardest label.

## AI Tool Plan

### Label Stress-Testing

Before annotating 200 examples, I will give Claude my three label definitions and the edge case decision rules above, and ask it to generate 5-10 synthetic r/nba posts that sit at the boundary between two labels. If I cannot cleanly classify the generated posts using my definitions, I will tighten the definitions before committing to full annotation. This is the highest-leverage use of AI in this project — catching definition gaps before they propagate into 200 inconsistent labels.

### Annotation Assistance

I will NOT use an LLM to pre-label examples. Manual annotation keeps me close to the data and forces me to apply the decision rules consistently. Pre-labeling risks introducing a second annotator's interpretation of my definitions, which could add noise rather than reduce it.

### Failure Analysis

After training and evaluation, I will paste the list of misclassified examples into Claude and ask it to identify systematic patterns — for example, whether errors cluster around short posts, sarcastic tone, a specific label pair, or posts that reference multiple events. I will then verify each claimed pattern by re-reading the flagged examples myself before including any pattern in the evaluation report. This helps surface patterns that are hard to see when reviewing errors one at a time.
