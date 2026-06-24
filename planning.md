# TakeMeter Planning Document

A fine-tuned text classifier that sorts Fire Force discourse into three quality categories: analysis, hot_take, and reaction.

## 1. Community

I chose the Fire Force fan community, mainly the Fire Force episode discussion threads on r/anime plus the standalone sub r/fireforce. I just finished the show, so the discourse is fresh in my head and I can label like an insider.

This community is a good fit because Fire Force fans argue about real, concrete things. They debate the animation quality and studio choices, the manga versus anime differences, the pacing of the later arcs, the fanservice running gag, the Soul Eater shared universe reveal, and the ending. That gives a real spread of take quality, from quick emotional reactions in episode threads to detailed breakdowns in theory and discussion posts. The variety is what makes the classification task interesting instead of trivial.

## 2. Labels

Three labels, mutually exclusive most of the time, each expected to clear 20 percent of the dataset.

**analysis**: A claim backed by something specific and checkable, such as animation or studio choices, pacing, a manga versus anime comparison, or a named scene or character beat.
- Example: "The anime compressed the Nether arc into too few episodes, so the reveal lands flatter than it does in the manga where the buildup is paced out."
- Example: "David Production reused a lot of CG for the fire effects in season 2, which is why the big fights look stiffer than the season 1 hand drawn ones."

**hot_take**: A bold, confident opinion stated with no real support. The claim might be true, but the post asserts instead of arguing.
- Example: "Fire Force is the most underrated shonen ever made and it is not even close."
- Example: "Season 3 is a complete waste of time, skip it and read the manga."

**reaction**: An in the moment emotional response to an episode or moment, with little to no argument.
- Example: "That finale absolutely wrecked me, I am not okay right now."
- Example: "Shinra going full Adolla burst gave me chills, replayed it five times."

## 3. Hard Edge Cases

The hardest case is the ending discourse, because the same topic can land in all three labels depending on how it is framed.

- "The ending was rushed and lazy." This is a hot_take. Bold quality claim, no support.
- "The ending felt rushed because the anime crammed the last two manga arcs into a handful of episodes." This is analysis. The claim is backed by a specific, checkable structural reason.
- "I cannot believe it is over, I am genuinely devastated." This is reaction. Pure feeling, no argument.

**Decision rule**: If removing the opinion still leaves real supporting evidence, label it analysis. If a detail is just decoration to sound credible, label it hot_take. If there is no argument at all and it is pure feeling, label it reaction.

Second edge case, the one detail hot take: "Season 2 is garbage, they ruined the animation." It points at a real thing, the animation, but it is opinion first and the detail is decorative, not argued. This goes to hot_take by the rule above.

I will keep a running notes column in the CSV and log every post that made me hesitate, then revisit them as a batch so I label similar cases the same way.

### Three hard cases encountered during annotation

1. **The ending split.** "The ending was rushed and lazy" versus "the ending felt rushed because they crammed the last manga arcs into too few episodes." Same topic, opposite labels. The first asserts with no support, so hot_take. The second gives a specific, checkable structural reason, so analysis. Decision: if removing the opinion still leaves real evidence, label analysis.

2. **The calm unsupported claim.** "I felt Joker's arc is unfinished." This is the trickiest case because the calm, measured tone reads like analysis. But it offers zero evidence for why the arc is unfinished, it just states it. Decision: tone does not earn the analysis label, only evidence does, so this is hot_take. This case sharpened my rule that analysis is about supporting evidence, not about how thoughtful a post sounds.

3. **The mixed structural complaint plus raw emotion.** "I'm legit going crazy over how no one brings up this disturbing design and how it never comes back." This blends a genuine structural observation (a design that appears once and is never explained) with an emotional outburst. Decision: when a post mixes registers, label it by the dominant one. Here the dominant register is the emotional reaction, not a built out argument, so reaction.

## 4. Data Collection Plan

Source: public posts and comments only. Three places to keep the labels balanced.
- r/fireforce for theory and discussion posts, which is where most analysis lives.
- The Fire Force episode discussion threads on r/anime, which is where most reaction lives.
- The season finale and series finale threads, which tend to be the richest mix of analysis and hot_take.

Target: at least 200 examples, aiming for a rough balance with no label above 70 percent and ideally each label at or above 20 percent. Realistic split target is around 70 analysis, 65 hot_take, 65 reaction.

If a label is underrepresented after 200, the likely shortfall is analysis, since episode threads skew toward reaction. Fix: go back to r/fireforce theory posts and the finale threads and pull more analysis specifically until each label clears 20 percent.

## 5. Evaluation Metrics

Accuracy alone is not enough, because a model can score decently on accuracy while completely failing one label if the classes are uneven. I need to see per class performance.

- **Overall accuracy** for both the baseline and the fine tuned model, as the headline number.
- **Per class precision, recall, and F1**, because the real question is whether the model learned all three boundaries or just the easy ones. F1 is the most useful single number per label.
- **Confusion matrix**, because it shows the direction of the errors. If the model keeps predicting hot_take when the truth is analysis, that tells me exactly which boundary it never learned, which is more useful than a single accuracy number.

The boundary I most expect to be hard is analysis versus hot_take, so I will watch those two cells of the confusion matrix closely.

## 6. Definition of Success

Good enough for a real community tool: overall accuracy clearly above the zero shot Groq baseline, and every per class F1 at or above 0.70. That threshold means the model learned all three distinctions, not just the dominant one.

Stretch success: analysis F1 at or above 0.75, since analysis versus hot_take is the boundary that actually matters for surfacing high quality posts in a community.

Floor I would not deploy below: any single class F1 near 0, which would mean the model cannot tell that label apart at all.

## AI Tool Plan

**Label stress testing**: Before annotating, I gave Claude my three label definitions and the ending edge case, and asked it to generate borderline posts that sit between two labels. Where it produced posts I could not cleanly classify, I tightened the definitions. This happened before locking the taxonomy.

**Annotation assistance**: I may paste batches of real posts I collected into Claude and ask it to pre label them using my definitions. If I do this, I will review and correct every single pre assigned label myself, because skimming defeats the purpose and produces noisy data. Any pre labeled batch is disclosed in the README AI usage section.

**Failure analysis**: After fine tuning, I will paste my list of wrong predictions into Claude and ask it to find patterns, such as a specific label pair that keeps getting confused, short posts, or sarcasm. Then I will re read the examples myself to verify each pattern before writing it up.

## Groq Baseline Prompt (for Section 5)

Paste this into the baseline cell, with each test post inserted where marked. It outputs only the label name so the notebook parser stays clean.

```
You are classifying a single post from the Fire Force anime community into exactly one of three labels.

analysis: a claim backed by something specific and checkable, such as animation or studio choices, pacing, a manga vs anime comparison, or a named scene or character beat.

hot_take: a bold, confident opinion stated with no real support. The claim may be true, but the post asserts instead of arguing.

reaction: an in the moment emotional response to an episode or moment, with little to no argument.

Rules:
- Choose exactly one label.
- If a post cites specific verifiable evidence, prefer analysis.
- If a post makes a bold quality claim with only decorative or no evidence, prefer hot_take.
- If a post is pure feeling with no argument, prefer reaction.

Output only the single label name, lowercase, nothing else.

Post:
"""{POST_TEXT}"""
```
