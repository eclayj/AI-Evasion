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

## Section 3 of 12 — Spam Filter Implementation

### Why this section builds a target model

GoodWords needs a classifier whose behavior can be measured. This section builds a **Multinomial Naive Bayes** spam filter from SMS messages. The model sees a numeric word-count representation, not the message's meaning as a person would understand it. The later attack can therefore change the counted evidence without changing the trained model.

This section reuses Bayes' rule and the conditional-independence assumption from Section 2. The new work is the practical pipeline: load examples, preserve useful text signals, turn messages into count vectors, train the classifier, and check its performance. We add a Module 318 notebook as a reusable baseline for this module. The existing Module 292 notebook teaches preprocessing and model-selection ideas with a different pipeline and vectorizer configuration, so it remains a useful reference but is not silently substituted for this module's target model.

### Bayes classifier: compare class scores

For a message `D`, Bayes' rule is:

\[
P(C\mid D)=\frac{P(D\mid C)P(C)}{P(D)}.
\]

Say it aloud: **“Probability of class C given document D equals probability of D given C, times probability of C, divided by probability of D.”** `C` is spam or ham; `P(C)` is the class prior; `P(D|C)` is how likely the message is under that class; and `P(C|D)` is the posterior after seeing the message. The denominator `P(D)` is shared by both classes, so to select a class the model can compare `P(C)P(D|C)` and omit that common denominator.

The predicted class is:

\[
\hat C=\operatorname*{arg\,max}_{c\in\{\mathrm{spam},\mathrm{ham}\}}P(c\mid D).
\]

Read `C-hat` as **“C hat,”** the predicted class; read `arg max` as **“the argument that maximizes.”** In English: return the label, spam or ham, whose posterior score is larger. `arg max` returns the label, not the numeric score itself.

The Naive Bayes approximation is:

\[
P(D\mid C)\approx\prod_{i=1}^{n}P(w_i\mid C).
\]

Say it: **“Probability of D given C is approximately the product from i equals one to n of probability of w sub i given C.”** `w_i` means the word-feature at position `i`; `n` is the number of features; and `\prod` means multiply the terms. If three word-count features have illustrative class-conditional probabilities `0.5`, `0.2`, and `0.1`, their simplified product is `0.01`. In reality, words influence one another; “naive” names the model's simplifying conditional-independence assumption, not a claim about how language works.

The implementation uses a log-score equivalent:

\[
\log P(C)+\sum_{i=1}^{n}\log P(w_i\mid C).
\]

Say it: **“log probability of class C plus the sum from i equals one to n of log probability of w sub i given C.”** The log identity `log(a × b) = log(a) + log(b)` converts the product into a sum. This avoids floating-point underflow from multiplying many tiny numbers. Because logarithm is increasing, the class with the larger probability still has the larger log-score.

### From text to a model input

`CountVectorizer` learns a vocabulary and represents each message by counts. For vocabulary `[` `free`, `hello` `]`, the message `free free hello` becomes `[2, 1]`: two occurrences of `free`, one of `hello`. These values are model features; they do not capture syntax, intent, or whether words fit together naturally.

The HTB page cleans cautiously so candidate signals survive: word case is normalized, Unicode and whitespace are standardized, while numbers, currency characters, and repeated punctuation can be retained. The section's custom token pattern explicitly recognizes words, currency, numbers, and repeated punctuation, and limits the vocabulary to 3,000 features. A character surviving text cleaning is not automatically a model feature; the vectorizer's token rules decide what becomes a column.

The dataset is imbalanced: HTB reports 747 spam and 4,825 ham messages in the original 5,572-row dataset. After its cleaning and duplicate removal, the displayed run has 5,153 messages, 638 spam (12.4%) and 4,515 ham (87.6%). A stratified 80/20 split in that run gives 4,122 training rows and 1,031 held-out test rows. In our end-to-end notebook run, the current archive URL supplied 5,574 rows (4,827 ham and 747 spam), and the same cleaning retained 5,155; the split was 4,124 training and 1,031 test rows. This two-row difference may reflect an archive or parsing revision. Keep the page's values as HTB's example and record the notebook's measured values as local results, not as a guaranteed match. `stratify=y` aims to preserve class proportions in both splits; `random_state=42` makes that particular random split reproducible.

The notebook deliberately learns the vocabulary only from the training messages and applies the learned vocabulary to test messages. Fitting preprocessing on all rows can leak information about the test set into training. The notebook also avoids saving or loading Python `pickle` files: untrusted pickle files can execute code when loaded, and keeping the fitted objects in memory is enough for a study baseline. These are practical safeguards around HTB's classroom flow, not changes to the GoodWords mathematics.

### Reading the evaluation

HTB's displayed run reports training accuracy `0.9922` and test accuracy `0.9864`, with spam precision `0.95` and recall `0.94`. These are run-specific example values, not guaranteed outputs; dataset version, library versions, split, and tokenization can change them.

Accuracy is the fraction of all predictions that are correct. Because ham is about 87.6% of this cleaned dataset, a model that always predicts ham would already achieve about 87.6% accuracy while catching no spam. For the spam class:

- **Precision** = true spam predictions / all predictions labeled spam. If 20 messages are flagged and 18 are spam, precision is `18/20 = 90%`.
- **Recall** = true spam predictions / all actual spam messages. If 24 spam messages exist and 18 are caught, recall is `18/24 = 75%`.
- **F1** is the harmonic mean of precision and recall; it is high only when both are reasonably high.

Precision answers “when the filter says spam, how often is it right?” Recall answers “of all spam, how much did it catch?” Both matter more than accuracy alone when the classes are imbalanced.

### Connection to the attack and COAE recall

The classifier turns a message into word-count features, then accumulates class evidence. GoodWords exploits that representation: words with stronger ham than spam evidence can move the ham score upward. The exact trained vocabulary and word probabilities matter, so later code should reuse this fitted baseline rather than casually training a different model.

**COAE quick recall:**

- Multinomial Naive Bayes scores word-count evidence by class; it is fast and interpretable, but the independence assumption is simplified.
- A bag-of-words vector is a row of feature counts. `CountVectorizer` learns the vocabulary; `transform` applies that already-learned vocabulary.
- Split before fitting the vocabulary or model; stratify to preserve the label mix; evaluate on held-out rows.
- With class imbalance, accuracy alone can hide failure on the minority class. Read spam precision and recall separately.
- The Module 318 [GoodWords baseline notebook](GoodWords_Baseline_Study.ipynb) implements only this section's dataset-to-classifier pipeline. The existing Module 292 [SMS preprocessing notebook](../AI_Applications/SMS_Spam_Preprocessing_Study.ipynb) remains a reference for text preprocessing, vectorization, and leak-aware model selection; its settings and fitted model are not assumed to match this module's baseline.

## Section 4 of 12 — GoodWords Attack Implementation

### White-box access to the model

This section assumes **white-box access**: the attacker can inspect the trained classifier's vocabulary and learned class-conditional word probabilities. A black-box attacker would instead need to infer useful additions from the model's visible input/output behavior. HTB extracts word statistics from the Naive Bayes model, ranks words that favor ham, appends the highest-ranked ones to held-out spam messages, and measures how often the predicted class changes.

The classifier stores log probabilities in a matrix:

\[
\texttt{feature\_log\_prob\_[c,i]}=\log P(w_i\mid c).
\]

Say the notation aloud as **“feature log probability at class c, feature i equals log probability of word w sub i given class c.”** The first index selects a class; the second selects a vocabulary feature. In scikit-learn, use `classifier.classes_` to look up which row corresponds to `ham` and `spam`. Hard-coding row zero as ham and row one as spam only works if the class ordering happens to match that assumption.

### Rank features by their relative ham evidence

The course converts a log probability back to a probability with the exponential and computes:

\[
S(w)=\frac{P(w\mid\text{ham})}{P(w\mid\text{spam})+\epsilon}.
\]

Say it: **“S of w equals probability of w given ham divided by probability of w given spam plus epsilon.”** `\exp(log p)` (“e to the log p”) recovers `p`; `\epsilon` is epsilon, a tiny positive constant that prevents division by zero. With `P(w|ham)=0.003`, `P(w|spam)=0`, and `epsilon=10^{-10}`, the score is about 30 million. The ratio is finite, but extremely large because the word was unseen in spam. That can make a rare word look spectacular despite weak absolute evidence.

Therefore, goodness is a **ranking score, not a probability or a guarantee of naturalness**. Inspect the underlying ham/spam probabilities and whether words make sense in context. The page itself notes that equal ratios can hide very different absolute frequencies: `0.005/0.0001=50`, and `0.5/0.01=50`, but the second word has much more support in the data. Another numerically safer ranking is the log-ratio `log P(w|ham) - log P(w|spam)`; the notebook preserves HTB's ratio formula so the course implementation is visible.

### Append candidates and test the class decision

For each tested spam message, the implementation appends the top `k` ranked words, then applies the already-fitted vectorizer and classifier. Reusing the fitted vectorizer is essential: it ensures each token maps to the same feature column the classifier learned. The raw message string must be used for the append operation; a sparse vector of counts cannot be concatenated with ordinary text.

The binary decision can be written:

\[
P(\text{ham}\mid D_{\text{augmented}})>P(\text{spam}\mid D_{\text{augmented}}).
\]

Say it: **“Probability of ham given the augmented document is greater than probability of spam given that augmented document.”** `D_augmented` means the original message with selected words appended. For a normalized two-class prediction, the two probabilities sum to one, so ham greater than spam is equivalent to ham probability greater than `0.5`. A result `[0.501, 0.499]` crosses the boundary just as `[0.99, 0.01]` does; the first barely flips, the second is much more confident.

The evasion rate is:

\[
\text{evasion rate}=\frac{\text{number of tested spam messages predicted ham}}{\text{number of spam messages tested}}\times100\%.
\]

Read the fraction as **“evaded divided by total tested, times one hundred percent.”** In HTB's display, `8/128 × 100 = 6.25%` evaded before any words were added. At 5 additions it reports `53/128 = 41.41%`; at 10, `95/128 = 74.22%`; at 15, `123/128 = 96.09%`; at 20, `128/128 = 100%`. These are results from HTB's displayed run and setup. Our notebook may differ because it trains a fresh model and split.

The zero-word result is not attack success: it records the model's natural false negatives on the test spam messages. Comparing later rates against this baseline shows the increase associated with the additions.

### Why the curve can rise quickly

For a message, the ham log-score contains a sum of per-word evidence:

\[
\log P(\text{ham}\mid D)=\log P(\text{ham})+\sum_{w\in D}\log P(w\mid\text{ham})-\log P(D).
\]

Say it: **“log probability of ham given D equals log probability of ham, plus the sum over words w in D of log probability of w given ham, minus log probability of D.”** That last normalization term is shared across the competing classes, so it cancels when the classifier compares their scores. This is the log-space view: every added feature contributes another term. HTB describes the resulting success curve as sigmoid-like because scores are converted back into probabilities. That shape is an intuition, not a guaranteed theorem: the notebook measures a finite set of spam messages at discrete word counts, so its plotted rate is a stepped empirical curve and may not be smooth or monotonic at every point.

### What the visualization communicates

Plotting `num_words` against `evasion_rate` shows how the measured attack success changes as the number of appended features grows. Horizontal 50% and 90% guides are visual reference levels; they do not represent universal security thresholds. The important measurement is the actual test-set count behind each percentage. A finite test set also limits precision: with 128 examples, one message changes the rate by about `100/128 ≈ 0.78` percentage points.

### COAE quick recall

- **White-box** means access to internal model details; in this section, the vocabulary and feature log probabilities.
- `feature_log_prob_[class, feature]` is a log conditional probability. Exponentiating recovers a probability; `classes_` should determine class-row indices.
- Goodness compares ham word probability with spam word probability plus epsilon. It ranks relative evidence and can overrate rare words.
- Augment raw text, then run the unchanged vectorizer and classifier. Do not append strings to count vectors.
- Binary evasion occurs when the ham probability exceeds spam probability. The evasion rate is a percentage over the tested spam subset; the zero-word measurement is the baseline false-negative rate.
- Treat plotted HTB percentages as experiment-specific. A sigmoid-like shape is a description of the observed trend, not a guaranteed curve.
- The Module 318 [GoodWords baseline notebook](GoodWords_Baseline_Study.ipynb) now continues with this section's white-box ranking, message augmentation, evasion measurements, and plot. It deliberately uses dynamic class indices and the raw `test_text` values so the code matches the actual variable shapes.

## Section 5 of 12 — Attack Analysis and Visualization

### What this analysis adds

Section 4 measured how many messages evade at each augmentation size. This section looks inside that aggregate result: it estimates each candidate word's individual effect and tracks spam probabilities for a few example messages as more words are appended. It reuses the model, vocabulary, top-word ranking, and held-out spam text already prepared in the Module 318 notebook.

### Individual impact and average impact

Let `f(x)` denote the model's spam probability for message `x`. After adding candidate word `w`, the course writes the impact as:

\[
I(w,x)=f(x)-f(x\oplus w).
\]

Say it aloud as **“impact of word w on message x equals f of x minus f of x combined with w.”** `x` is the original message; `w` is the candidate word; `\oplus` means the message with that word appended; and `f` returns a probability between 0 and 1. A positive result means the spam probability went down. For example, if it changes from `0.95` to `0.85`, then `I=0.95-0.85=0.10`.

The average across `N` sampled messages is:

\[
\overline{I}(w)=\frac{100}{N}\sum_{i=1}^{N}\left[f(x_i)-f(x_i\oplus w)\right].
\]

Read the notation aloud as **“average impact of w equals one hundred divided by N, times the sum from i equals one to N of spam probability for x sub i minus spam probability for x sub i with w appended.”** `N` is the number of sampled spam messages; `x_i` is the message at sample index `i`; `\sum` means add each message's change. Multiplying by 100 expresses the average probability change in **percentage points**. A drop from 0.95 to 0.85 is 10 points; it is not a 10% relative reduction. The relative reduction would be `0.10/0.95 ≈ 10.5%`.

HTB tests the top 20 ranked features on 50 spam messages and reports the leading word, `lor`, reducing spam probability by roughly 4 percentage points on average in its run. These values depend on the trained model and message sample.

### Goodness is not the same as measured impact

Section 4's goodness ratio compares estimated word probabilities in ham and spam across the training data. Individual impact measures a word's change to `f(x)` on the particular held-out messages. They answer different questions. A word can be strongly ham-associated in aggregate, yet lower spam probability only modestly when the original message contains many strong spam features.

Impact also depends on the baseline message. Some words may affect a promotional message differently from a short phishing-like message. Averaging over examples reduces dependence on one message but does not make the estimate universal. The first 50 rows of a fixed split are a course convenience; sampling rules and split randomness can affect the result.

### Probability-shift plot

For several messages, the notebook measures `P(spam | message)` at addition counts such as `0, 5, 10, 20, 30`, then plots grouped bars against the decision boundary. Say `P(spam | message)` as **“probability of spam given the message.”** A bar below `0.5` is classified as ham in this binary setting. If a message starts below `0.5` at zero additions, it was already a false negative; added words did not cause that initial error.

The plot is a per-message view of the same decision process as Section 4's aggregate evasion rate. It shows why a single average can hide variation: some samples cross after a few additions, some need more, and some are already misclassified before the attack.

### Important math caveat: changes are not generally additive in probability space

The Naive Bayes **log scores** add per-feature evidence, but the normalized class probabilities are nonlinear functions of those scores. Thus, if one word lowers spam probability by four points on average, ten words do not necessarily lower it by forty points. Word effects can overlap, probabilities are bounded between zero and one, and normalization changes how each shift appears. The correct approach is to measure combined words directly, as Section 4 did.

HTB uses “percentage reduction” in prose, but its subtraction `prob_orig - prob_aug`, followed by multiplication by 100, is an absolute difference in **percentage points**. The notebook labels it that way for precision.

### COAE quick recall

- `I(w,x)=f(x)-f(x⊕w)` measures absolute change in spam probability after appending one word. Positive means lower spam probability.
- Average impact is the mean of that change over `N` messages, multiplied by 100 to display percentage points.
- Goodness ranks a word from training-set class probabilities; impact measures its effect on particular test messages. A high ratio does not imply a large effect.
- A grouped probability plot shows differences between messages and marks the 0.5 binary decision boundary. Baseline false negatives must be separated from attack-induced flips.
- Log evidence accumulates additively, but probability reductions do not generally add linearly. Measure word combinations rather than extrapolating from a single-word average.
- The [GoodWords baseline notebook](GoodWords_Baseline_Study.ipynb) now includes single-word impact analysis, an impact bar chart, and per-message probability shifts while reusing the existing model and test-text series.

## Section 6 of 12 — Black-Box GoodWords Attack

### How black-box changes the problem

Sections 4–5 used **white-box access** to inspect the Naive Bayes vocabulary and feature probabilities. In a **black-box** setting, the attacker sees only submitted messages and returned labels or confidence scores. It cannot read model weights, feature distributions, architecture, or training data. Each query may cost time, money, or detection risk, so the attacker must learn efficiently from a limited number of observations.

### Unknown classifier and constrained word choice

HTB models the classifier as:

\[
f:\mathcal{X}\rightarrow[0,1].
\]

Read it aloud: **“f maps calligraphic X to the interval zero to one.”** `\mathcal{X}` (“calligraphic X”) is the set of possible messages; `f` maps a message to its spam probability, between 0 and 1. A transformation `T:\mathcal{X}\rightarrow\mathcal{X}` (“T maps message space to message space”) edits an input while keeping it a message. The goal is to lower `f(T(x))` while preserving the original message's meaning.

The selection objective is:

\[
\min_{\substack{W\subseteq\mathcal{V}\\|W|\leq k}} f(x\oplus W).
\]

Say it aloud: **“Minimize, over sets W that are subsets of calligraphic V and whose size is at most k, f of x combined with W.”** `\min` means find the smallest objective value; `W` is the selected word set; `\mathcal{V}` (“calligraphic V”) is the candidate vocabulary; `|W|` (“the size of W”) counts selected words; `k` is the maximum allowed additions; and `\oplus` (“combined with” or “appended to”) adds those words to message `x`. In ordinary English: choose no more than `k` candidate words that make the black-box classifier return the lowest spam probability.

### Estimate a word's effect with queries

Without access to gradients or feature probabilities, submit the original and modified messages and compare their scores:

\[
r_w(x)=f(x)-f(x\oplus\{w\}).
\]

Read it aloud: **“Reward for word w on message x equals f of x minus f of x combined with the set containing w.”** `r_w(x)` is the reward; `f(x)` is the original spam score; `x\oplus\{w\}` is the message with word `w` appended; and `{w}` is a one-word set. If the score moves from `0.95` to `0.80`, reward is `0.15`; if it moves to `0.94`, reward is `0.01`. Positive reward means the word lowered the spam score.

For a set `W`, the analogous reward is `r_W(x)=f(x)-f(x\oplus W)`. These are **finite-difference measurements**: compare outputs for two nearby inputs instead of computing a derivative. The idea connects to Section 5's probability impact, but here the model's internal probabilities are unavailable and each measurement consumes a query.

### Exploration versus exploitation

Treat each candidate word as a **bandit arm** (one slot machine with an unknown payout). Testing the word uses a query; the returned score change is its observed reward.

- **Exploration:** test a word with little evidence to find out whether it could be better than current favorites.
- **Exploitation:** keep using words with high observed average reward.

Only exploiting known winners can miss a better word. Only exploring new words spends the query budget without using what has already been learned. A **multi-armed bandit** strategy balances both choices.

### Upper Confidence Bound (UCB)

The page's worked numbers are consistent with this UCB score:

\[
\operatorname{UCB}(w)=\bar r_w+\sqrt{\frac{c\ln(t)}{n_w}}.
\]

Read it aloud: **“U-C-B of w equals average reward r-bar sub w, plus the square root of c times natural log of t divided by n sub w.”** `\bar r_w` (“r-bar sub w”) is the average observed reward for word `w`; `t` is the total number of queries so far; `n_w` (“n sub w”) counts how many times word `w` has been tested; `c` controls exploration strength; and `\ln` is natural logarithm. The **first term** rewards words that have worked well. The **second term** is an uncertainty bonus: fewer tests mean a larger bonus.

For HTB's illustrative setup, use `c=2` and `t=200`:

- “thanks”: average reward `0.15`, tested `50` times: `0.15 + sqrt(2 ln(200)/50) ≈ 0.61`.
- “appreciate”: average reward `0.12`, tested `5` times: `0.12 + sqrt(2 ln(200)/5) ≈ 1.58`.

The less-tested “appreciate” receives the larger UCB score, so the algorithm tries it to reduce uncertainty—even though its observed average is lower. HTB's page rounds the intermediate bonuses and totals to about `0.47`/`0.62` and `1.48`/`1.60`; with a calculator and `ln(200)≈5.298`, the formula gives about `0.46` and `1.46`. The small difference is rounding in the course example; the exploration-versus-exploitation conclusion is unchanged.

An untested word has `n_w=0`; the formula would divide by zero. A real implementation handles this explicitly by testing unseen candidates first (equivalently, treating their exploration bonus as infinite), then applies UCB after every candidate has at least one observation.

### Guarantees and practical limits

HTB cites **logarithmic regret**, `O(log T)`. Read this as **“order of log T.”** `T` is the number of queries; regret is the accumulated reward lost compared with always choosing the best option in hindsight. Logarithmic growth is slower than linear growth in query count. Classical UCB1 guarantees require rewards to be independent, stationary (their distribution does not change over time), and bounded. Real word effects depend on the message and on other appended words, so those guarantees do not automatically apply to every GoodWords experiment. UCB is a principled query-allocation strategy, not a promise of optimal attack success.

The page sketches a black-box workflow: build a candidate vocabulary offline, estimate reward adaptively, keep moving averages as new observations arrive, then test combinations for possible synergy. A single-word reward is only a first-order effect; combinations may interact, so measure the combined score instead of assuming individual rewards add exactly.

### COAE quick recall

- **White-box:** inspect internal model data. **Black-box:** query inputs and observe outputs, with the model internals hidden.
- The objective selects `W⊆V`, `|W|≤k`, to minimize the returned spam score for an augmented message.
- `r_w(x)=f(x)-f(x⊕{w})` is an output difference measured by queries; a positive reward means lower spam score.
- A bandit arm is a candidate word. Exploration tests uncertain choices; exploitation reuses words with good measured rewards.
- UCB combines average reward with an uncertainty bonus `sqrt(c ln(t)/n_w)`. Test every untried word explicitly before using the formula.
- `O(log T)` regret relies on stationary, independent, bounded rewards; message- and combination-dependent word effects weaken those assumptions.
- **Notebook decision:** this page presents a strategy but no runnable code. Keep the Module 318 notebook's current white-box implementation separate; do not claim that its access to `feature_log_prob_` is black-box. A black-box implementation belongs in a later subsection if HTB introduces runnable query code and a concrete query contract.

## Section 7 of 12 — Black-Box Core Components

### What this subsection prepares

Section 6 introduced the query-only threat model and UCB search. This subsection builds the reusable inputs for that search: a candidate vocabulary drawn from legitimate training messages, a small curated supplement, a query-budget tracker, and a 40/40/20 planning split. It does **not** yet query a hidden classifier or select attack words from model responses; those steps belong to later material.

### Count candidate words from ham messages

For a candidate word `w`, the notebook's frequency is:

\[
\operatorname{freq}(w)=\sum_{m\in H}\operatorname{count}(w,m),
\]

where `H` is the sampled set of ham messages and `count(w,m)` is how many times `w` appears in message `m`. Read it aloud as **“frequency of w equals the sum, over ham messages m, of the number of times w occurs in m.”** This counts token occurrences, not the number of distinct messages containing the token. Repeating a word twice in one message adds two.

The code uses the first 500 ham training messages, splits each on whitespace, and keeps tokens whose lengths are 3 through 9 characters. For instance, `hi` is removed, while `thanks` is kept. It then selects up to 100 tokens with frequency **greater than** five. That strict `>` means a word must occur at least six times; a parameter named `min_freq=5` can otherwise be misread as “five or more.” Candidate ranking is descending frequency, with alphabetical ordering to break ties reproducibly.

The sample is taken in existing row order rather than randomly. That is a simple course implementation, but it can make the vocabulary sensitive to dataset ordering. A production or research experiment would usually sample reproducibly or use a separate permitted corpus, normalize/tokenize consistently with the model, and examine document frequency as well as raw counts. Most importantly, this vocabulary-building step assumes access to legitimate example text. A strictly query-only attacker with no such corpus must obtain candidates from another allowed source.

### Curated terms and final ordering

The course merges frequent ham tokens with a small hand-picked conversational list using a set union. A set union removes duplicates; sorting then makes output deterministic. Curated candidates absent from the ham sample have frequency zero, so the final helper places them after frequency-backed words and orders ties lexicographically. The curated list hedges against a small sample missing useful everyday language; it does not establish that those words help the hidden target. Their usefulness must be measured with queries.

### Query budget arithmetic

The subsection initializes `B=1000` allowed model queries and `q=0` queries used. It reserves 40% for exploration, 40% for exploitation, and the remainder for combinations:

\[
B_{\text{explore}}=\lfloor0.4B\rfloor,\quad
B_{\text{exploit}}=\lfloor0.4B\rfloor,\quad
B_{\text{combine}}=B-B_{\text{explore}}-B_{\text{exploit}}.
\]

Read this as **“exploration budget equals the floor of 0.4 times total budget; exploitation budget is the same; combination budget is total budget minus those first two.”** The floor symbol means round down to an integer. With `B=1000`, the allocations are 400, 400, and 200; they sum to 1000. Putting the remainder in the last phase also makes the total exact for budgets that do not divide evenly.

The ratio is a planning heuristic, not a mathematical optimum. The query log is initialized for later measurements; no budget is consumed merely by creating it. Any real oracle call should be checked against the remaining budget before submission and counted exactly once afterward. Failed, repeated, or batched requests also need a clearly defined accounting rule matching the target service.

### Why the existing notebook variables need care

The baseline notebook's `X_train` is a sparse numeric count matrix after vectorization. Word mining needs the original message strings, so Section 7 uses `train_text` with the aligned `y_train` labels. This distinction matters: splitting the sparse matrix yields feature-column numbers, not the original tokens. The held-out spam strings are copied into a separate list before shuffling, so this setup does not reorder the earlier notebook's `spam_test_text` series.

### COAE quick recall

- Candidate mining counts tokens from ham text; in this implementation it is **token frequency**, so repeats count multiple times.
- `len(w)>2 and len(w)<10` keeps lengths 3–9. `count > min_freq` with `min_freq=5` means six or more.
- The top-100 cap applies before merging curated words; the final list can exceed 100. Set union removes duplicates, and deterministic tie-breaking supports reproducibility.
- Use raw training text (`train_text`) for token mining. `X_train` is already a sparse feature matrix and no longer contains directly readable messages.
- For a 1,000-query budget, the course's 40/40/20 split gives 400 exploration, 400 exploitation, and 200 combination queries. This is a heuristic; the remainder rule keeps totals exact.
- Vocabulary construction and budget initialization do not themselves call the black-box model. Section 7 is setup, not attack execution.
- The training-text vocabulary assumes a legitimate corpus is available. A query-only adversary without one needs another source; candidate selection still does not prove target effectiveness.
- The [GoodWords baseline notebook](GoodWords_Baseline_Study.ipynb) adds this setup with visible alignment checks, deterministic shuffling, and explanations. It does not reuse white-box scores to rank these black-box candidates.

## Section 8 of 12 — Black-Box Adaptive Discovery Methods

### From a candidate list to adaptive experiments

Section 7 prepared candidate words and a finite query budget. This section tracks what each tested word appears to do, decides whether to try a new or promising word, and checks whether selected pairs work better together. The same bandit pattern from Section 6 is present, but HTB now demonstrates **epsilon-greedy** selection and an **exponential moving average** instead of UCB.

### Epsilon-greedy: explore sometimes, exploit most of the time

The selection rule uses a fixed exploration rate `ε=0.2`. Say `ε` as **“epsilon.”** With probability 0.2, choose an untested or least-tested candidate; otherwise choose the candidate with the best current score. A random number drawn uniformly from 0 to 1 is compared with epsilon: if it is less than 0.2, explore. For example, a draw of 0.08 explores; a draw of 0.64 exploits.

Exploration protects against overlooking a better word whose early results are not known. Exploitation spends most queries reusing observed winners. This rule differs from Section 6's UCB: UCB assigns every word a score containing an uncertainty bonus, while epsilon-greedy uses a fixed probability to switch into exploration. A fixed epsilon is easy to explain, but may keep exploring at the same rate indefinitely. Under standard stationary bandit conditions it can have linear worst-case regret; UCB's logarithmic-regret result requires its own assumptions. An epsilon schedule that decays over time is one alternative, but HTB uses a fixed rate for this example.

### Exponential moving average

HTB updates a word estimate with:

\[
S_t=(1-\alpha)S_{t-1}+\alpha r_t.
\]

Read aloud: **“S sub t equals one minus alpha times S sub t minus one, plus alpha times r sub t.”** `S_t` is the new score estimate; `S_{t-1}` is its prior value; `r_t` is the latest measured reward; and `\alpha` (“alpha”) is the learning rate. With `\alpha=0.3`, 70% of the new estimate comes from the old score and 30% from the latest reward. If the old score is `0.10` and the latest reward is `0.15`, then `S_t = 0.7(0.10)+0.3(0.15)=0.115`.

The count map separately records how often each word has been tested. The EMA weights recent observations more heavily than distant ones without storing every result, which is memory efficient and can adapt when conditions shift. But it is not the ordinary arithmetic mean: order matters, and it is not automatically an unbiased estimate. With a fixed message, repeated results from a deterministic classifier may be identical; to learn behavior across messages, measure multiple messages and be clear whether the baseline is cached or queried again.

### Pair synergy and limited search

Let `I(w)` be a word's impact on a fixed message, and `I(w_1,w_2)` the impact of the pair. HTB defines a pair's excess over the separate impacts as:

\[
\operatorname{synergy}(w_1,w_2)=I(w_1,w_2)-\left(I(w_1)+I(w_2)\right).
\]

Say it aloud: **“synergy of w one and w two equals their combined impact minus the impact of w one plus the impact of w two separately.”** If each word alone reduces spam probability by `0.10`, while the pair reduces it by `0.25`, the computed synergy is `0.25-(0.10+0.10)=0.05`, a positive, super-additive result on that message. A negative value indicates their observed combined reduction is less than the sum of the separate reductions.

This is a measured heuristic in probability space, not a claim that classifier internals or probabilities are additive. Scores are bounded from 0 to 1 and normalized nonlinearly; if the separate impacts sum to more than the available baseline probability, the formula can even give a negative synergy despite a useful combined attack. Always keep pair impact and pair synergy as distinct quantities.

For `n` candidates, the number of unique pairs is “n choose 2,” written `\binom{n}{2}` and pronounced **“n choose two”**; the count is `n(n-1)/2`. For triplets it is “n choose 3,” `\binom{n}{3}=n(n-1)(n-2)/6`. With 20 words, exhaustive search has 190 pairs and 1,140 triplets, in addition to 20 single words. HTB reduces cost by testing all 105 pairs from 15 candidates and extending only five leading positive-synergy pairs with up to 10 words. This is a bounded, greedy search: it can miss a useful pair or triplet outside those shortlists.

### A model wrapper is not automatically a black box

The HTB example directly calls `vectorizer.transform(...)` and `classifier.predict_proba(...)`. That is convenient for simulating returned scores using the local training notebook, but it exposes the model and so is **white-box access** in the literal threat-model sense. The notebook places these operations behind `query_spam_probability(message)`, which returns only a score to the adaptive functions and counts each submission. This creates a query-shaped interface while still using the local classifier underneath. A real black-box client must replace that wrapper body with an authorized service request and follow that service's authentication, request format, rate limits, and query-accounting rules.

The budget helper estimates the worst-case requests for the course search before it starts. For 20 singles, 105 pairs, and at most 50 triplets, this is at most 176 queries including the original-message baseline. Smaller shortlists reduce that cost. The wrapper refuses to exceed the remaining budget rather than silently sending an extra query. In this notebook the demonstration uses only five words for combination search to stay compact.

### COAE quick recall

- Epsilon-greedy explores with fixed probability `ε`; otherwise it exploits the best current score. `ε=0.2` means 20% exploration and 80% exploitation.
- `S_t=(1-α)S_{t-1}+αr_t` is an exponential moving average: `α` controls the weight of the newest observation. It differs from a plain sample mean and is order-sensitive.
- Pair synergy is measured combined impact minus the two single impacts. Positive is super-additive for that sampled input; it does not prove a general interaction in the model.
- “n choose k” counts combinations without regard to order. Exhaustive pair/triplet search grows rapidly, motivating HTB's shortlist-based greedy search.
- The local `classifier.predict_proba` call is wrapped to imitate a score-only query, but the underlying demonstration still has white-box access. Replace the wrapper to connect a true black-box service.
- Query-budget checks should happen before each request. The current demo records one unit per submitted message and reserves enough budget before running combinatorial search.
- The [GoodWords baseline notebook](GoodWords_Baseline_Study.ipynb) now includes adaptive selection, EMA score updates, and a budgeted pair/triplet search with an explicit local-simulator versus real black-box distinction.

## Section 9 of 12 — Black-Box Discovery Algorithm

### Information gain: the reason to spend queries carefully

HTB introduces query value with:

\[
I(Q)=H(W)-H(W\mid Q).
\]

Read aloud: **“information from query Q equals entropy of W before the query minus entropy of W given the query result.”** `W` stands for uncertainty about which candidate words are effective. `H` (“entropy”) measures uncertainty in a distribution; `H(W)` is the uncertainty before observing the query and `H(W\mid Q)` is the remaining uncertainty after its result. Subtracting tells us how much uncertainty the observation removed.

For example, if four candidates are equally plausible, their uncertainty is `log₂(4)=2` bits. If a result narrows the possibilities to two equally plausible candidates, the remaining uncertainty is `log₂(2)=1` bit, so one bit of uncertainty was removed. This is an illustrative case, not a prediction that one word query will split the candidate list evenly.

The course formula is useful intuition, but the formal expected information gain of choosing query action `q` averages over its possible outputs `Y`:

\[
\operatorname{EIG}(q)=H(W)-\mathbb{E}_{Y\mid q}[H(W\mid Y,q)].
\]

Say it aloud: **“expected information gain of query q equals the current entropy minus the average remaining entropy over the possible results Y of q.”** The distinction is that the outcome is not known before sending the query. HTB's implementation does not calculate entropy or expected information gain; it uses fixed budget phases and empirical score reductions as a practical heuristic.

### Three phases and their query arithmetic

The budget split is 40% exploration, 40% exploitation, and 20% combinations. With 1,000 total model submissions, those phase budgets are 400, 400, and 200. A single-word experiment compares the original message with its augmented version, so it costs two submissions. Therefore 400 exploration queries fund 200 original/modified comparisons, not 400 experiments. The same arithmetic applies to exploitation.

1. **Exploration:** sample across messages and candidate words; use epsilon-greedy selection to find useful candidates. HTB tracks 25%, 50%, and 75% milestones.
2. **Exploitation:** focus on stronger word estimates, with a smaller exploration rate, to refine them. HTB narrows to the leading candidates and a smaller message sample.
3. **Combination search:** compare individual impacts with pairs and test a limited set of triplets. For the section's 20-candidate cap, the worst-case local helper costs one baseline plus 20 singles, 105 pairs, and 50 triplets: 176 messages per input. Three inputs fit within the 200-query phase budget at most when the shortlist is reduced; our notebook chooses a per-input shortlist that fits and counts each submitted message through the same wrapper.

The notebook's third phase may leave part of the allocated 200 queries unused because it stops after at most three inputs and the combination search is bounded. Staying within budget is more important than forcing the counter to equal 1,000. The course page reports exactly 1,000 partly by estimating combination cost as twice the number of results; the notebook accounts for actual score requests instead.

### What each phase learns

In exploration, a reward is still the finite difference `f(x)-f(x⊕{w})`: original spam probability minus probability after appending word `w`. The adaptive scorer updates its EMA and query counter. The top-scoring candidates—not every word—then feed exploitation and combination testing. Each phase narrows effort, so early noise can influence which words are examined later; the result is a heuristic search, not a proof that the global best combination was found.

The section describes information gain but does not estimate uncertainty distributions. Epsilon-greedy rewards are based on observed changes, and the phase budgets are fixed. This differs from a full Bayesian experimental-design system, which would maintain a posterior over word effects and select queries by expected uncertainty reduction.

### COAE quick recall

- `I(Q)=H(W)-H(W|Q)` is the section's uncertainty-reduction intuition. Formal expected information gain averages remaining entropy over possible outputs of a chosen query.
- `H(W)=-Σ p(w) log₂ p(w)` means **“entropy of W equals the negative sum, over candidates w, of probability p of w times base-two log of p of w.”** For four equally likely candidates, entropy is 2 bits.
- A one-word impact comparison costs two requests: original plus modified. Thus 400 phase queries yield 200 paired experiments.
- The 40/40/20 split for 1,000 requests means 400 exploration, 400 exploitation, 200 combination. It is a heuristic allocation, not an information-theoretic optimum.
- Use the same query wrapper and shared counter across phases; do not maintain a local count that drifts from the requests actually sent.
- The course's `predict_proba(...)[0][1]` assumes a class column order and exposes internals. The notebook routes scores through the existing wrapper, which counts each message and uses the dynamically determined spam index. The wrapper still simulates a black box locally.
- Phase 3 should account for baseline, individual, pair, and triplet requests, not estimate them from the number of returned dictionary entries. A query cap and shortened shortlist prevent overruns.
- The [GoodWords baseline notebook](GoodWords_Baseline_Study.ipynb) now includes the complete phased workflow, exact request accounting, query milestones, and combined/individual impact summaries.

## Section 10 of 12 — Black-Box Attack Implementation

### From discovered words to a measured evasion rate

This section applies the discovered word ranking to spam messages and compares black-box discovery with the earlier white-box GoodWords ranking. The black-box selection signal is empirical impact, `r_w(x)=f(x)-f(x⊕{w})`: measure the original spam score, append word `w`, and subtract the new score. This differs from white-box goodness, which ranks words from learned class-conditional probabilities. The two rankings can disagree because one measures training-set association and the other measures observed changes on particular messages.

For `N` evaluation messages and a chosen word set, the attack reports:

\[
R=100\times\frac{E}{N},
\]

where `E` is the count whose augmented message has spam probability below the `0.5` decision threshold. Read it aloud as **“R equals one hundred times E divided by N.”** For example, `14` evaded messages out of `20` gives `100×14/20=70%`. The percentage is only meaningful alongside its numerator and denominator. If only a few messages remain when a budget runs out, report that reduced sample size; do not present it as though all planned examples were tested.

The zero-word row measures the baseline false-negative rate: messages already classified as ham before modification. Comparing `R(k)` with `R(0)` helps distinguish those pre-existing errors from additional messages flipped by the attack. A result at `k` words is not necessarily the same as the rate of attack-caused flips unless the baseline cases are accounted for.

### Keep discovery and evaluation separate

The Academy sample selects `spam_messages[:50]` for discovery, then `spam_test_messages[30:50]` for evaluation. Those ranges overlap on messages 30–49, so they are not a fresh holdout despite the prose. The notebook instead uses the first 50 test spam messages for discovery and the next 20 for evaluation. A truly independent final estimate would ideally use a separate validation set or freshly collected examples; this split is a course-study workaround.

Evaluation also consumes queries. Seven augmentation sizes on 20 messages require up to `7×20=140` submissions. The notebook reserves those 140 requests first and gives the three-phase search the remaining 860 from the total 1,000-query budget. The phase allocator then works on that reduced budget, and each actual request still goes through the shared counter. White-box comparison scores are computed locally from the same evaluation messages and are not counted as black-box service requests; they are a separate local control.

### Course code differences worth recognizing

The page's sample calls `build_candidate_vocabulary(X_train, y_train)`, but in our notebook `X_train` is already a sparse feature matrix. Candidate mining needs the original strings in `train_text`. The sample also calls the classifier directly, assumes spam is probability column `[1]`, estimates combination queries after running them, and contains summary names that are not defined in this notebook (`test_acc`, `word_impacts`, `output_dir`). The notebook reuses the Section 9 result dictionary, dynamic `spam_index`, exact query wrapper, and existing `word_impact_df`, then defines a results directory and JSON import when saving.

The course page's example outputs (for example, “really” at `0.121` impact or a typical 85–95% of white-box performance) are experiment-specific claims, not guaranteed outcomes. Impact depends on the archive version, preprocessing, model, random seed, test split, sample size, and candidate vocabulary. Performance percentages should be computed from this run's rates rather than copied from the lesson.

### COAE quick recall

- `r_w(x)=f(x)-f(x⊕{w})` measures the score decrease; goodness from `P(w|ham)/P(w|spam)` is a different, white-box ranking criterion.
- `R=100×E/N`: evasion rate is a percentage over measured examples. Always retain `E` and `N` and report the baseline zero-word rate.
- The page's `30:50` evaluation range overlaps its first 50 discovery samples. Use disjoint examples when estimating generalization.
- Reserve evaluation queries before allocating the discovery budget. In this notebook, 20 examples × 7 word counts = 140 reserved, leaving 860 for discovery out of a 1,000 request total.
- Count every original, augmented, and evaluation submission through one wrapper. Respect the configured budget rather than incrementing an approximate counter afterward.
- Access probability columns by their class label (`spam_index`), not an assumed column position. In this notebook the local model is still only a simulator for a black-box interface.
- The 85–95% performance comparison and specific word impacts are not universal. Recompute them on the actual model and a disjoint evaluation sample.
- Results persistence should serialize plain Python numbers and words. Store query totals, sample denominators, thresholds, and configuration with the rates so results remain interpretable.

## Section 11 of 12 — GoodWords Challenge

### What the challenge asks

The challenge gives a fixed high-confidence spam message and a frozen Multinomial Naive Bayes service. We may only **append** legitimate-looking letter tokens, stay within the service's `max_added_words` limit, and make the returned label become ham. The challenge is black-box: `/predict` exposes a label and spam probability, but not the model's parameters. A valid `/submit` returns the flag.

The API sequence is:

- `GET /health`: check that the instance responds.
- `GET /challenge`: retrieve the fixed base message, target label, and word budget.
- `POST /predict`: ask for the score of an original or augmented message.
- `POST /submit`: send the final append-only text and receive a flag if validation succeeds.

The base message is selected by the server and validated to be strongly spam under its frozen model. Candidate vocabulary from the earlier lesson is only a starting point; the challenge model may use different training data or tokenization, so measure effects against this service.

### Finite-difference ranking

For base message `x` and candidate word `w`, the notebook measures:

\[
\Delta p(w)=p_{\text{spam}}(x)-p_{\text{spam}}(x\mathbin{+}\text{space}+w).
\]

Read it aloud as **“delta p of w equals spam probability for x minus spam probability for x followed by a space and w.”** `p_spam(x)` is the returned spam score for the unmodified message. A positive `Δp(w)` means that one appended word lowered the score; for example, `0.91-0.76=0.15`, a 15 percentage-point decrease. This is the finite-difference reward from Section 6 measured directly through `/predict`, with each request consuming time and potentially service query allowance.

Ranking single-word effects is a useful first filter, but effects depend on context. After adding one word, the next word can have a different marginal effect than it had by itself. The challenge scaffold therefore checks the label after each growing prefix and stops as soon as it becomes ham. The endpoint's label is the authoritative success signal; do not assume a local classifier's class ordering or threshold is identical.

### Append-only validation and token budget

The submission must begin with the exact base message followed by a space and added tokens. Count the **additional tokens** using the challenge's letter-only tokenizer; punctuation inside an added item may split it into multiple tokens. The notebook restricts added candidates to `[A-Za-z]+`, joins them with single spaces, verifies that the original base is unchanged, and recounts the suffix before submission. These checks prevent a visually small suffix from exceeding the actual token budget.

For a budget `B`, never append more than `B` validated tokens. If the API says `B=25`, a candidate sequence of 26 words is invalid even if the classifier already predicts ham at word 20; stop at the first successful prefix and submit that prefix.

### Security concepts and practical handling

- **Oracle/query interface:** the service answers chosen inputs without revealing internal parameters. It is a black-box classifier from the attacker's perspective.
- **Frozen model:** one active challenge instance returns scores from a fixed model. Starting a fresh instance may change its target URL or state; get the current constraints again.
- **Rate limiting and network reachability:** the target may only be reachable through the active HTB instance/VPN. Use timeouts and HTTP status checks; do not treat a connection failure as a classifier result.
- **Flag handling:** `/submit` should only receive a final candidate that passed local append-only/token-budget validation and whose latest prediction is ham. The response flag is the challenge artifact; keep it out of the reusable study notes and notebooks.

### COAE quick recall

- The attack is black-box and query-based: compare `/predict` outputs rather than inspect model internals.
- `Δp(w)=p_spam(x)-p_spam(x+w)` is the single-word finite-difference reward. Positive means a lower spam score, not automatically a successful label change.
- Append-only means the original base text stays byte-for-byte intact; add a separator and legal tokens after it.
- Count tokens with the challenge's letter-only rule, not whitespace items or characters. Respect the server-provided maximum.
- Re-score the growing prefix and stop when the service reports the target label. Word impact is context-dependent, so a rank is a heuristic.
- Keep `BASE_URL` configurable for the active target. The challenge notebook is a client scaffold; it contains no persistent target IP or flag.

### What the live run demonstrated

The spawned service returned a baseline spam probability of about `0.9999999983`. Individual candidate words changed that score only by tiny amounts; many scores displayed as `1.0` at four decimal places. Yet after 14 appended words, the service returned ham with spam probability about `0.2041`, below the 25-token budget. The notebook used 55 `/predict` requests: one baseline, 40 single-word probes, and 14 checks of the growing prefix. This is a useful example of probability saturation and context dependence: single-word rankings were weak clues, while the cumulative sequence produced a clear class change. The target address and flag are intentionally omitted from this reusable study guide.

## Section 12 of 12 — Skills Assessment: Feature Obfuscation Attack

### Two phases, two access levels

The assessment asks for 10 sentiment flips in each of two modes:

1. **White box:** start with 10 positive reviews, inspect the downloaded classifier, and append at most 30 words per review to make each prediction negative.
2. **Black box:** start with 10 negative reviews, hide the model internals, and use only `/predict` outputs to append at most 40 words per review until each prediction is positive.

Both phases must pass before the assessment service returns its flag. An appended word is a model feature: it changes the evidence the classifier accumulates, even when it does not alter the original review's wording.

### White-box word evidence

For a token `w`, the white-box notebook ranks its class evidence with:

\[
L(w)=\log P(w\mid\text{negative})-\log P(w\mid\text{positive}).
\]

Read aloud: **“L of w equals log probability of w given negative minus log probability of w given positive.”** `P(w|negative)` is the classifier's learned probability for the token in negative reviews; the positive class is analogous. A positive `L(w)` means the token favors negative sentiment over positive sentiment in the model's learned likelihoods. For example, if the two probabilities are `0.02` and `0.001`, then `L=log(0.02/0.001)=log(20)≈3.00`, a strong negative direction.

This ranking comes from the model's internal `feature_log_prob_` values, so it is appropriate only for Phase 1. The assessment vectorizer includes multi-word features; adding one word can also create new adjacent n-grams. Therefore the unigram likelihood ratio is a candidate-ranking shortcut, not a complete prediction. The downloaded classifier verifies every growing review locally, using its actual vectorizer and class labels.

The model bundle is a pickle. Standard `pickle.load` can execute arbitrary globals embedded in the file. The assessment notebook permits only the specific NumPy and scikit-learn objects observed in the HTB bundle, loads it in memory, and rejects every other global. This narrows the code that can be reconstructed while keeping model analysis local.

### Black-box word impact

For a negative review `x` and word `w`, the black-box phase measures:

\[
\Delta p_{+}(w,x)=p_{+}(x\oplus w)-p_{+}(x).
\]

Read aloud: **“delta p positive of w and x equals positive probability after appending w minus positive probability before.”** `p_+(x)` is the API's returned positive-class probability. If it rises from `0.18` to `0.34`, the measured increase is `0.16`, or 16 percentage points. Ranking candidates by this change is like the earlier GoodWords reward but with the target class reversed. Only the API label confirms a flip; a high positive probability estimate is still different from the server's final success decision.

### Budgets, evaluation, and accounting

Each review has its own added-word cap: 30 in Phase 1 and 40 in Phase 2. The notebook verifies the exact original prefix and counts each appended single-token word before submission. The challenge's JSON submissions must contain all 10 original IDs and their corresponding augmented texts. It reports per-review feedback, allowing failed cases to be refined without changing successful ones.

The black-box API's query allowance is distinct from the word cap. Each `/predict` request consumes a query; probing 30 candidates for 10 reviews is already 300 requests, before testing prefixes. The notebook uses a fixed, editable candidate list and sequential requests; reduce the probe list if the service imposes tighter query limits. It never uses Phase 1's model weights to rank Phase 2 black-box candidates.

### COAE quick recall

- White-box target evidence: `log P(w|negative)-log P(w|positive)`. Positive favors the target negative class; it is a feature-ranking heuristic and exact predictions should be verified.
- Black-box target impact: `p_positive(augmented)-p_positive(original)`. Positive means the target-class score rose. Query outputs replace access to weights.
- `arg max` means **“the argument that maximizes”**: the word whose measured target-class impact is largest.
- The white-box download is a pickle. The notebook uses an allowlist-based unpickler and rejects unexpected globals instead of calling unrestricted `pickle.load`.
- Per-review limits are 30 and 40 added words. Count only appended tokens and preserve each original text exactly.
- Both phases require all 10 review IDs with the corresponding target sentiment. Check server feedback and final `/status`; one phase alone does not complete the assessment.
- Keep white-box and black-box data paths separate. Do not reuse downloaded weights to select words in the black-box phase.


### Assessment result and exam takeaways

Both phases completed successfully: all 10 white-box reviews were flipped within the 30-word cap, and all 10 black-box reviews within the 40-word cap. The black-box run made 1,177 `/predict` requests during candidate probing and prefix checks. Per-review additions ranged from 3 to 39 words. These counts describe this live instance; they are not fixed requirements or guarantees for another instance.

This run is a useful exam example of **score saturation**. Some original positive-class probabilities were extremely close to zero, so one-word probes could produce changes too small to notice or rank reliably. A sequence of individually measured words changed the model input cumulatively; the service label and final submission response determined whether the attack succeeded. Single-word impact is a local measurement, not a promise that the same word keeps the same impact after other words are appended.

A query budget and an appended-word budget are separate constraints. A candidate probe asks the service a question but does not have to appear in the final text; an appended word changes the review but does not itself consume a query unless the API is called again to measure the result. Keep an explicit counter for each resource. The assessment API's `/status` confirmed 10 of 10 completions in each phase.

For test review, explain why the two modes use different evidence: the white-box likelihood ratio reads model parameters, while the black-box finite difference measures API outputs. Both still need an authoritative final classifier label, strict preservation of the original review, and compliance with the per-review word cap.
