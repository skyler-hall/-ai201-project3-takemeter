# Loom video link (also in tyhe notes of submission) : https://www.loom.com/share/03f4cda79b4b4c9795243e8a799c8ca5
# TakeMeter: Fire Force Discourse Classifier

A fine tuned DistilBERT model that classifies posts from the Fire Force anime community into three discourse quality labels: analysis, hot_take, and reaction. This README documents the build, the evaluation, and an honest analysis of why the fine tuned model underperformed a zero shot baseline.

## Community Choice and Reasoning

I chose the Fire Force fan community, drawing from r/fireforce and the Fire Force episode and finale discussion threads on r/anime. I just finished the show, so I can label like a regular. The community argues about concrete things, animation quality, manga versus anime differences, pacing, the fan service running gag, the Soul Eater prequel reveal, and the ending, which gives a real spread of take quality from quick emotional reactions to detailed breakdowns. That variety is what makes the classification task meaningful instead of trivial.

## Label Taxonomy

**analysis**: A claim backed by something specific and checkable, such as animation or studio choices, pacing, a manga versus anime comparison, or a named scene or character beat.
- Example 1: "The anime compressed the Nether arc into too few episodes, so the reveal lands flatter than the manga."
- Example 2: "Tamaki only has stripping scenes for about 6 minutes out of a runtime of 300+ minutes."

**hot_take**: A bold, confident opinion stated with no real support.
- Example 1: "Genuinely the best ending of any modern shounen imo."
- Example 2: "A lot of people are sleeping on Fire Force, I'm gonna keep saying it."

**reaction**: An in the moment emotional response with little to no argument.
- Example 1: "That finale wrecked me, I am not okay."
- Example 2: "Damn near fell off the chair seeing that for the first time, it blew my mind."

## Data Collection and Labeling

**Source**: 204 public posts and comments collected manually from r/fireforce and the Fire Force episode and finale threads on r/anime. I pulled lore and theory posts for analysis, top opinion posts for hot_take, and episode discussion threads for reaction.

**Labeling process**: I read each post and applied the definitions above. I used an LLM to pre label batches and to clean formatting, then reviewed and corrected every single label myself. Ads, memes, joke chains, non English comments, and a small number of comments rationalizing the sexualization of an underage character were excluded.

**Label distribution** (204 total):
- analysis: 84 (41 percent)
- hot_take: 61 (30 percent)
- reaction: 59 (29 percent)

No label exceeds the 70 percent ceiling and every label clears 20 percent.

**Three difficult to label examples and my decisions**:
1. "The ending was rushed and lazy" versus "the ending felt rushed because they crammed the last manga arcs into too few episodes." Same topic, but the first is hot_take (no support) and the second is analysis (specific structural reason). Decision rule: if removing the opinion still leaves real evidence, it's analysis.
2. "I felt Joker's arc is unfinished." This reads thoughtful and calm, so it's tempting to call it analysis, but it offers zero evidence for why. I labeled it hot_take, because the calm tone does not change that it asserts without arguing.
3. "I'm legit going crazy over how no one brings up this disturbing design and how it never comes back." This mixes a real structural complaint with raw emotion. I labeled it reaction, because the dominant register is the emotional outburst, not a built out argument.

## Fine Tuning Approach

**Base model**: distilbert-base-uncased from HuggingFace.

**Training setup**: Fine tuned on the 70 percent train split (about 142 examples) in Google Colab on a T4 GPU. The data was split 70/15/15 into train, validation, and test, stratified by label.

**Hyperparameter decision**: I kept the notebook defaults of 3 epochs, learning rate 2e-5, and batch size 16. With only 142 training examples I chose not to increase epochs, because more passes over such a small set risks overfitting to the majority class rather than learning the harder boundaries. As the results show, even at 3 epochs the model still collapsed toward the majority class.

## Baseline Description

**Prompt used**: Zero shot classification with Groq llama-3.3-70b-versatile. A system prompt gave the three label definitions and rules and instructed the model to output only the lowercase label name. The post itself was passed in a separate user message. Full prompt is in planning.md.

**How results were collected**: The notebook ran the prompt over all 31 test examples, parsed the single label response, and computed accuracy and per class metrics on the same test set used for the fine tuned model. All 31 responses parsed cleanly.

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
| --- | --- |
| Groq zero shot baseline | 0.645 |
| Fine tuned DistilBERT | 0.419 |

The fine tuned model regressed 0.226 below the baseline. This is the opposite of the usual result and is the central finding of this project.

### Per Class Metrics (Fine Tuned DistilBERT)

| Label | Precision | Recall | F1 | Support |
| --- | --- | --- | --- | --- |
| analysis | 0.41 | 0.92 | 0.57 | 13 |
| hot_take | 0.50 | 0.11 | 0.18 | 9 |
| reaction | 0.00 | 0.00 | 0.00 | 9 |

### Per Class Metrics (Groq Baseline)

| Label | Precision | Recall | F1 | Support |
| --- | --- | --- | --- | --- |
| analysis | 1.00 | 0.62 | 0.76 | 13 |
| hot_take | 0.43 | 0.33 | 0.38 | 9 |
| reaction | 0.56 | 1.00 | 0.72 | 9 |

The baseline spreads predictions across all three labels. The fine tuned model does not, which is clearest in the confusion matrix below.

### Confusion Matrix (Fine Tuned)

Rows are true labels, columns are predicted. Supplementary image is confusion_matrix.png.

| true \ pred | analysis | hot_take | reaction |
| --- | --- | --- | --- |
| analysis | 12 | 1 | 0 |
| hot_take | 8 | 1 | 0 |
| reaction | 9 | 0 | 0 |

Of 31 test posts, the model predicted analysis 29 times, hot_take twice, and reaction never. It collapsed onto the majority class.

### Three Wrong Predictions Analyzed

1. **Post**: "Damn near fell off the chair seeing that for the first time, it blew my mind, so cool." **True**: reaction. **Predicted**: analysis. **Why it failed**: This is pure emotional reaction, but the model predicted reaction zero times across the entire test set. It never learned the label at all, so even an obvious reaction gets swept into analysis.
2. **Post**: "Genuinely the best ending of any modern shounen imo." **True**: hot_take. **Predicted**: analysis. **Why it failed**: This is a bold claim with no support, a clean hot_take. The model only correctly caught 1 of 9 hot_takes. With about 42 hot_take training examples, it never learned to separate a confident bare opinion from an argued claim, so it defaults to analysis.
3. **Post**: "Goodbye peak anime." **True**: reaction. **Predicted**: analysis. **Why it failed**: A short emotional farewell. Short, low information posts give the model very little signal, and with analysis as the dominant class the safest guess is analysis. The model exploited that shortcut.

The directional pattern is unmistakable: almost every error is X predicted as analysis. The model learned one thing, predict the majority class, rather than the three boundaries I intended.

### Sample Classifications

| Post | Predicted | Confidence |
| --- | --- | --- |
| "The anime compressed the Nether arc into too few episodes, so the reveal lands flatter." | analysis | high |
| "Genuinely the best ending of any modern shounen imo." | analysis | moderate |
| "That finale wrecked me, I am not okay." | analysis | moderate |

The first prediction is reasonable and correct: the post cites a specific structural choice (compressing an arc) and explains its effect, which is exactly what the analysis label is meant to capture. The second and third are wrong, both swept into analysis, which illustrates the collapse described above.

(Note: replace "high" and "moderate" with the actual softmax confidence scores if your notebook prints them per prediction. If it does not, run the fine tuned model on these three strings and report the max softmax probability.)

## Reflection: What the Model Learned vs What I Intended

I intended the model to learn three boundaries: argued versus asserted (analysis versus hot_take) and emotional versus reasoned (reaction versus the rest). What it actually learned was a single decision rule: predict analysis. Because analysis was the largest class at 41 percent and the training set was tiny at about 142 examples, the cheapest way to minimize training loss was to lean on the majority class rather than learn the genuinely hard, subjective distinctions. It overfit to class frequency, not to meaning. It missed the entire reaction class (F1 of 0.00) and nearly all of hot_take (recall 0.11).

The deeper lesson is about model and data fit. A subjective, nuanced task like take quality needs either far more labeled data or a model that already understands argumentation. The 70 billion parameter Groq model brings that understanding from pretraining and spreads its predictions sensibly with zero examples, which is why it won. DistilBERT, learning from scratch on 142 posts, had no such prior and defaulted to the shortcut. The fix would be more data, several hundred per class, class weighting or oversampling to counter the imbalance, or simply accepting that for this task and this data budget a zero shot LLM is the better tool.

## Spec Reflection

One way the spec helped: it insisted I run the zero shot baseline before fine tuning and on the same locked test set. Without that, I would have seen 0.419 accuracy and assumed the task was just hard. The baseline at 0.645 reframed the result entirely, it told me the task is learnable and that my fine tuned model specifically failed, which is a much more useful finding.

One way my implementation diverged: the spec frames the baseline as the thing to beat, with the implied happy path being that fine tuning wins. My implementation diverged from that expectation because the baseline won. Rather than treat this as a bug to fix, I followed the spec's own guidance that a diagnosable failure is more valuable than a model that quietly works, and I made the regression the centerpiece of the analysis.

## AI Usage

1. **Annotation assistance and data cleaning**: I directed an LLM to pre label batches of posts I collected, using my label definitions, and to strip ads, usernames, vote counts, and non English text into clean rows. It produced suggested labels with one line reasons. I reviewed and overrode every label myself, including reclassifying several borderline cases (for example, calm but unsupported claims that I moved from analysis to hot_take). I also overrode its inclusion of a few comments and excluded content that rationalized sexualizing an underage character.
2. **Failure pattern analysis**: I pasted my per class metrics and confusion matrix into an LLM and asked it to identify the dominant error pattern. It surfaced the majority class collapse (almost all errors being X predicted as analysis, reaction never predicted). I verified this myself by reading the confusion matrix directly, where the 29 of 31 analysis predictions confirmed it, and I confirmed the reaction F1 of 0.00 in the per class table before writing it up.
