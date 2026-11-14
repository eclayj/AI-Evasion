# HTB Academy AI Red Teamer Path — Module 318: AI Evasion - Foundations

These notes follow the module in order and are intended as a study guide and COAE review reference. Review sections stay concise when they repeat ideas or code already covered elsewhere; new attacks and code receive fuller explanations. Course examples are distinguished from small teaching examples and production considerations.

## Section 1 of 12 — Introduction to AI Evasion Attacks

### The core idea: change the input at prediction time

An **evasion attack** changes what a trained model sees when it makes a prediction. The attacker does not need to change the training data, retrain the model, or edit its parameters. The aim is to craft one inference-time input so the model produces a wrong or attacker-preferred result.

This timing distinguishes evasion from nearby attack types:

| Attack | When it acts | What it changes |
| --- | --- | --- |
| Data poisoning | Training | Examples used to teach the model |
| Label manipulation | Training | The answers attached to training examples |
| Trojan/backdoor | Usually planted during training or model preparation; triggered at inference | A hidden trigger-to-behavior association |
| Evasion | Inference | The particular input being predicted |

For a **toy** bag-of-words spam filter, imagine its feature record is `[free=2, meeting=0]`. Add ordinary words such as “meeting” and “agenda,” and the record might become `[free=2, meeting=2]`. The model still has the same learned weights; the changed word counts can move its score toward “not spam.” A hypothetical score could move from 0.61 to 0.47 across a 0.50 decision threshold. Those numbers illustrate the idea; they are not HTB's measured result. To a person, the message may still plainly be spam.

A **decision boundary** is the learned dividing rule between predicted classes. The GoodWords attack makes the boundary visible in a familiar setting: carefully selected benign tokens shift the features that a probabilistic spam classifier uses, potentially changing which class wins.

### Transferability and black-box testing

**Transferability** means an input crafted against one model can sometimes fool another model. A **surrogate model** is a substitute used for experiments when the production model's internals are unavailable. Transfer is more plausible when the models learned similar feature patterns, but it is never guaranteed. In a **black-box** setting, an assessor may see only inputs and outputs through an API, perhaps with a query budget; they cannot inspect the deployed weights or code.

### How this connects to LLMs

Prompt injection also manipulates inference-time input, without changing the model's weights. The shared security idea is to change the input/context so the model behaves differently. The practical surface differs: a spam classifier maps structured word features to a label, while an LLM generates open-ended text that another application may display, parse, or turn into a tool action. This module starts with the more measurable GoodWords spam-filter example.

### COAE quick recall

- If training examples or labels are altered, classify the attack as training-time poisoning or label manipulation; if only the prediction input is crafted, classify it as evasion.
- A surrogate model enables offline experimentation, and transferability may carry a successful input to a similar target; it is not a promise of success.
- Evasion occurs at inference and leaves the trained parameters unchanged.
- A GoodWords-style attack changes benign-looking text features to shift a spam model's class score.
- **No equation or runnable code appears in this introduction.** The [Module 292 SMS study notebook](../AI_Applications/SMS_Spam_Preprocessing_Study.ipynb) already contains the earlier text pipeline and classifier workflow; later sections can build on that material when HTB introduces attack code.

## Section 2 of 12 — The GoodWords Attack

### Attack idea

The GoodWords attack appends words associated with legitimate mail to an otherwise unchanged spam message. It does not need to disguise or remove the original sales or phishing message. Instead, it changes the **features** the classifier counts. A Naive Bayes spam filter can be fooled because its simplified model adds each word's evidence as though the words were independent once the class is known; it does not understand that “meeting” and “thanks” may be semantically out of place in the message.

### Bayes' rule and the class decision

HTB's refresher is Bayes' rule:

\[
P(C\mid D)=\frac{P(D\mid C)\,P(C)}{P(D)}.
\]

Say it: **“Probability of class C given document D equals the probability of document D given class C, times the probability of C, divided by the probability of D.”** `C` is the candidate class, `D` is a document/message, `P(C)` is the **prior** before reading this message, `P(D|C)` is how likely this evidence is under that class, and `P(C|D)` is the updated **posterior** after seeing it. `P(D)` normalizes the result across possible classes.

For a toy numeric example, suppose 40% of messages are spam, and a particular message has likelihood 0.60 under spam versus 0.20 under ham. The total probability of observing it is `0.60×0.40 + 0.20×0.60 = 0.36`. Bayes' rule gives `P(spam|D) = 0.60×0.40/0.36 ≈ 0.67`; the ham posterior is about 0.33, so spam wins. The earlier [Module 292 Bayes notebook](../AI_Applications/Spam_Classification_Bayes_Study.ipynb) works through HTB's own two-feature numbers by hand and in Python.

The decision rule is:

\[
\text{class}=\operatorname*{arg\,max}_{c\in\{\text{spam},\text{ham}\}}P(c\mid D).
\]

Read `arg max` aloud as **“the argument that maximizes.”** Here it means: “return whichever class label `c`—spam or ham—has the larger posterior probability.” `arg max` returns the winning **label**; `max` by itself would return the winning score. If spam has probability 0.67 and ham 0.33, the returned class is spam.

### The “naive” conditional-independence assumption

HTB factorizes a document likelihood as:

\[
P(D\mid C)=\prod_{i=1}^{n}P(w_i\mid C).
\]

Say it: **“Probability of document D given class C equals the product, from i equals one to n, of probability of w sub i given C.”** `w_i` (“w sub i”) is the word-feature at index `i`; `n` is the number of word-features; `\prod` (“the product”) means multiply all the terms. The model assumes the word-features are conditionally independent **given the class**. For two features whose probabilities under spam are 0.5 and 0.4, this approximation gives a joint likelihood of `0.5×0.4=0.20`.

This is a simplifying model assumption, not a claim that human language has no word relationships. A bag-of-words classifier does not model phrase meaning or whether the full message makes sense. That makes it computationally tractable, but creates an opening: each added feature can contribute more evidence for ham even when it conflicts with the message's intent.

### Log space turns products into sums

Multiplying many probabilities smaller than one can produce extremely tiny numbers. The classifier can instead compare log-scores:

\[
\text{class}=\operatorname*{arg\,max}_{c\in\{\text{spam},\text{ham}\}}
\left[\log P(c)+\sum_{i=1}^{n}\log P(w_i\mid c)\right].
\]

Read it: **“Choose the class c that maximizes log probability of c plus the sum, from i equals one to n, of log probability of w sub i given c.”** `\log` is “log”; `\sum` is “the sum”; the square brackets group the score being compared. The logarithm identity `log(a×b)=log(a)+log(b)` changes a long product into a sum. Because log is increasing, the class with the larger probability also has the larger log probability. The common denominator `P(D)` from Bayes' rule can be omitted while comparing classes because it is the same for every candidate class.

### Original and augmented message scores

Let the original spam message be `M_spam = {w_1, …, w_m}`, and the added good-word set be `G = {g_1, …, g_k}`. Say `w_i` “w sub i,” `g_j` “g sub j,” and `m`/`k` as “m”/“k,” the counts of original and added word-features. The augmented message is written `M_augmented = M_spam ∪ G`; say **“M augmented equals M spam union G.”** Here union is the course's notation for appending the extra word-features. In an actual bag-of-words implementation, repeated words may be represented as counts, so the exact feature update depends on the vectorizer.

For the original message, HTB writes:

\[
\log P(\text{spam}\mid M_{\text{spam}})
=\log P(\text{spam})+\sum_{i=1}^{m}\log P(w_i\mid\text{spam}),
\]
\[
\log P(\text{ham}\mid M_{\text{spam}})
=\log P(\text{ham})+\sum_{i=1}^{m}\log P(w_i\mid\text{ham}).
\]

Say the first equation: **“Log probability of spam given M spam equals log probability of spam plus the sum, i equals one through m, of log probability of w sub i given spam.”** The second is the same calculation for ham. The first term on each right side is that class's prior score; the sum is the evidence from the message words.

After adding `G`, the spam score gains `\sum_{j=1}^{k}\log P(g_j|spam)` and the ham score gains `\sum_{j=1}^{k}\log P(g_j|ham)`. A word associated more with legitimate mail adds a larger ham-side contribution. Success means:

\[
\log P(\text{ham}\mid M_{\text{augmented}})
>\log P(\text{spam}\mid M_{\text{augmented}}).
\]

Read the `>` symbol as “is greater than.” In plain English, the augmented message receives a higher ham score than spam score, so the model changes its predicted class.

The page rearranges that comparison into a useful threshold condition:

\[
\sum_{j=1}^{k}
\left[\log P(g_j\mid\text{ham})-\log P(g_j\mid\text{spam})\right]
>
\sum_{i=1}^{m}
\left[\log P(w_i\mid\text{spam})-\log P(w_i\mid\text{ham})\right]
+\left[\log P(\text{spam})-\log P(\text{ham})\right].
\]

Say it: **“The sum over j of each good word's log-probability given ham minus its log-probability given spam must be greater than the sum over i of each original word's spam-versus-ham log-probability difference, plus the prior spam-versus-ham log-probability difference.”** Each bracketed difference is a relative advantage. The **left side** is the added words' total advantage for ham. The **right side** is the spam evidence already present in the original message plus any prior bias toward spam. The attack needs enough added ham advantage to exceed that whole hurdle.

HTB's small example gives the arithmetic directly: if the original message and prior create an 8-unit hurdle toward spam, and each selected good word contributes 3 units toward ham, three words add `3×3=9` units—enough to cross the boundary. This example uses abstract score units to make the inequality visible; they are not measured probabilities from the live HTB model.

### Ranking words with a goodness score

HTB ranks candidate words with:

\[
S(w)=\frac{P(w\mid\text{ham})}{P(w\mid\text{spam})+\epsilon}.
\]

Say it: **“S of w equals probability of w given ham, divided by probability of w given spam plus epsilon.”** `S(w)` is a **ratio score**, not a probability; `\epsilon` (“epsilon”) is a small positive constant added to the denominator to avoid division by zero. Higher scores indicate that the word is relatively more associated with ham than spam.

For example, if `P(w|ham)=0.45`, `P(w|spam)=0.05`, and `epsilon=0.01`, then `S(w)=0.45/(0.05+0.01)=7.5`. That word looks useful by this ratio. A ratio alone can overrate a word supported by very few observations, so HTB also calls for representative absolute frequency, natural-looking text, and consideration of other filters or message-length limits. Epsilon in this ranking formula is a numerical guard; it is not the same thing as the later Naive Bayes smoothing parameter.

### Course result versus cited research

The HTB page reports that its examples often evade with about 15–30 selected additions and over 90% success. Lowd and Meek's 2005 paper studied different filters, data, and attack settings: it reports that at most 150 added words in a **passive** attack (no feedback from the target filter), or 30 in an **active** attack (test queries allowed), could get half of currently blocked spam past either filter. Treat these as results from different experiments, not directly comparable guarantees. The research source is [“Good Word Attacks on Statistical Spam Filters”](https://www.ceas.cc/papers-2005/125.pdf).

### COAE quick recall

- GoodWords alters inference-time message features; it does not retrain or modify the model.
- The Naive Bayes conditional-independence assumption makes word-feature likelihoods multiply, and log space turns their contributions into a sum.
- A candidate word is useful when it adds more conditional log-probability evidence for ham than spam; the message flips when accumulated ham evidence beats the original spam evidence and prior.
- `arg max` gives the winning class label, while `max` gives only the largest score; `S(w)` is a ranking ratio, not a class probability.
- **Notebook decision:** This subsection presents the mathematics but no new code cell. Keep using the existing Module 292 [Bayes calculation notebook](../AI_Applications/Spam_Classification_Bayes_Study.ipynb) for the refresher and the [SMS pipeline notebook](../AI_Applications/SMS_Spam_Preprocessing_Study.ipynb) for the trained classifier. A GoodWords implementation belongs only in a later section that introduces it; none is pre-built here.
