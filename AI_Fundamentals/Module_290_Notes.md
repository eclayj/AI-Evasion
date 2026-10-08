# Fundamentals of AI — Module 290

These are original study notes for the HTB Academy AI Red Teamer path. They follow the course one section at a time and explain security connections for a learner comfortable with programming but new to AI security. A course concept is identified as such; a security implication or engineering caveat is labeled separately. No live target, token, or answer belongs in this file.

## Section 1 of 24 — Introduction to Machine Learning

### The three nested terms

| Term | Plain-English meaning | Example |
| --- | --- | --- |
| Artificial intelligence (AI) | The broad effort to make a system perform tasks associated with intelligence, such as interpreting language, recognizing objects, reasoning, or making decisions. | A rule-based troubleshooting expert system or an image-recognition service. |
| Machine learning (ML) | A way to build AI behavior by fitting a model to examples, rather than writing every decision rule by hand. | A spam filter trained on labeled messages. |
| Deep learning (DL) | A family of ML methods that uses neural networks with multiple layers to learn useful representations from data. | A convolutional network that learns visual patterns in images. |

Read the relationship as **deep learning is inside machine learning, and machine learning is inside AI**. The words are not interchangeable. A hand-written rule engine may be AI without being ML. A decision tree is ML without being deep learning. A neural network is not automatically *deep* just because it has the word “network” in its name; depth refers to multiple learned layers. The nested-circle picture in HTB is a teaching aid, not a claim that every intelligent behavior fits one fixed taxonomy.

For a data-engineering analogy, think of AI as the overall product capability (“classify suspicious events”), ML as a component that learns a mapping from historical records, and DL as one possible model family for that component. A production system may also contain non-ML rules, databases, APIs, human review, and monitoring. The model is not the whole application.

### Three ways a learning problem can provide feedback

1. **Supervised learning:** each training example includes a desired answer, or *label*. For email filtering, an example is a message plus “spam” or “not spam.” Training adjusts a model so its predictions better match those labels. Classification predicts a category; regression predicts a number. HTB introduces the category here and teaches individual algorithms later.
2. **Unsupervised learning:** examples arrive without those answer labels. The model looks for structure such as groups, unusual points, or lower-dimensional representations. A cluster of similar network events is a pattern, not automatically an attack. A human or a separate process must interpret what the pattern means.
3. **Reinforcement learning:** an *agent* chooses an action, the *environment* changes, and the agent receives a reward or penalty. Over many interactions it learns a *policy*—a rule for choosing actions in situations. This differs from being handed the correct action for every example. A game-playing agent learning from wins and losses is the course's intuitive example.

The easiest question to ask is: **Where does the feedback come from?** A supplied answer means supervised learning; no supplied answer means unsupervised learning; rewards after actions mean reinforcement learning. Real systems may combine these approaches, but keeping the basic cases separate will help when we reach their algorithms.

### What deep learning adds

Traditional ML often depends on people choosing useful input features—for instance, message length, sender reputation, and the presence of suspicious phrases. A deep network can learn intermediate features from rawer inputs. In an image system, early layers may react to edges or textures; later layers combine those responses into more complex shapes. “Early” and “later” describe processing stages, not a guarantee that every neuron has a neat human-readable meaning.

HTB names convolutional neural networks (CNNs) for images, recurrent neural networks (RNNs) for sequences, and transformers for language. Treat these as architecture families optimized for different data relationships, not as rules that restrict each family to only one task. “End-to-end” training means the final task's learning signal can adjust multiple stages together; it does not mean data collection, validation, deployment, and security happen automatically.

### Cybersecurity bridge (additional context)

- A **false negative** is a real threat that a detector misses. A **false positive** is benign activity that it flags. These errors have different costs: a missed intrusion can be damaging, while too many false alerts can overwhelm analysts.
- A model can learn only from the examples and labels it receives. An attacker who changes training records or labels may contaminate what it learns; that is a preview of **data poisoning**. An attacker who changes an input at prediction time to evade a trained detector is attempting **evasion**. Later path modules examine these attacks directly.
- Unsupervised anomaly detection asks “does this differ from the usual pattern?” It does **not** answer “is this malicious?” A new legitimate deployment can look anomalous, and a careful attacker can look ordinary.
- Reinforcement learning optimizes the stated reward. If the reward is a poor stand-in for the real safety objective, an agent may find a high-scoring but unwanted behavior. Security review must examine the reward definition and what the agent can control.

### Check your understanding

- A fixed list of hand-written fraud rules: potentially AI, but not ML just because it makes decisions.
- A decision tree trained on labeled fraud cases: ML and supervised learning, but not DL.
- Grouping unlabeled traffic records by similarity: unsupervised ML; the groups still need interpretation.
- A multi-layer neural network trained to identify objects in labeled images: supervised DL (and therefore also ML and AI).

There is no new mathematical notation or course code in this section. We will add equations and annotated notebook cells when the course introduces algorithms that need them, rather than pre-implementing later sections here.

## Section 2 of 24 — Mathematics Refresher for AI

HTB presents this page as a **reference**, not a test of mathematical fluency. We will return to these symbols when an algorithm needs them. The translation pattern below is: **say it aloud → identify each part → state the whole idea in plain English → calculate a small example**. In code blocks on the HTB page, `*` means multiplication; in formulas, a centered dot or adjacency may mean the same thing. A vector is an ordered list of numbers, and a matrix is a rectangular table of numbers.

### Addresses, powers, functions, and sums

- `$x_t$` is **“x sub t.”** The subscript `t` is an index or address, commonly a time step. If a sensor's readings are `[10, 13, 12]`, then with time steps `1, 2, 3`, `x_2 = 13`. A subscript is not multiplication by `t`. HTB's example also contains a vertical bar in `$q(x_t\mid x_{t-2})$`; read that bar **“given.”** It describes a quantity concerning the current value given a value two steps earlier. The page writes `$x_t=q(x_t\mid x_{t-2})$`, but those sides need not have the same type: a reading is a value, while `q(...)` may denote a probability or model output. Treat that line as an illustration of indexing, not a general identity.
- `$x^n$` is **“x to the n-th power.”** The superscript tells us to multiply `x` by itself `n` times when `n` is a positive integer. `$3^2=3\times3=9$` is “three squared equals nine.” Do not confuse a superscript with a subscript: `$x_2$` selects an item; `$x^2$` squares a value.
- `$f(x)$` is **“f of x.”** `f` is a function and `x` is its input. If `$f(x)=x^2+2x+1$`, then `$f(2)=2^2+2(2)+1=9$`. In programming terms, a function maps an input to an output; mathematical notation often leaves its implementation unspecified.
- `$\sum_{i=1}^{n} a_i$` is **“the sum from i equals one to n of a sub i.”** `i` is a loop counter, `a_i` is the item at index `i`, `1` is the first index, and `n` is the last. The whole expression means add every listed item. For `$a_1=2,a_2=4,a_3=6$`, `$\sum_{i=1}^{3}a_i=2+4+6=12$`. This is the mathematical version of a loop that accumulates a total.
- `$1/x$` is **“one over x,”** the reciprocal of nonzero `x`. `$1/5=0.2$`; division by zero is undefined. The dots in `$a_1+a_2+\cdots+a_n$` are read **“and so on”**—they indicate the same pattern continues, not an extra number.
- `max(4,7,2)=7` is **“the maximum of four, seven, and two is seven”;** `min(4,7,2)=2` is the corresponding minimum. The comparison signs `≥`, `≤`, `=`, and `≠` read **greater than or equal to**, **less than or equal to**, **equals**, and **not equal to**. Python writes these as `>=`, `<=`, `==`, and `!=` when testing values; `=` in Python is assignment, so do not copy mathematical equality syntax blindly into code.

### Measuring a vector's size: norms

Let `$v=(3,4)$`, a two-component vector. Bars around a number mean absolute value: `$|-4|=4$`. Double bars around a vector denote a *norm*, a rule that assigns it a nonnegative size.

- `$\lVert v\rVert_2=\sqrt{v_1^2+v_2^2}$` is **“the L-two norm of v equals the square root of v sub one squared plus v sub two squared.”** Each `v_i` is one component, squaring removes its sign and emphasizes larger components, the sum combines them, and the square root returns a length in the original units. For `(3,4)`, `$\sqrt{3^2+4^2}=\sqrt{25}=5$`. The complete idea is straight-line distance from the origin.
- `$\lVert v\rVert_1=|v_1|+|v_2|$` is **“the L-one norm of v equals the absolute value of v sub one plus the absolute value of v sub two.”** It totals the sizes of individual changes: `$|3|+|4|=7$`.
- `$\lVert v\rVert_\infty=\max(|v_1|,|v_2|)$` is **“the L-infinity norm of v equals the maximum absolute component.”** It measures the single largest component: `$\max(3,4)=4$`.

These are **different questions** about the same vector: total absolute movement, straight-line movement, and worst single-coordinate movement. In later evasion attacks, the vector may be a perturbation `$\delta=x'-x$`—read **“delta equals x-prime minus x”**—so choosing a norm changes what “small change” means. Note that *squared* L2, `$\lVert v\rVert_2^2=v_1^2+v_2^2=25$`, is **not** the L2 norm itself (`5`).

### Matrices without the mystique

For `$A=\begin{bmatrix}1&2\\3&4\end{bmatrix}$` and `$v=\begin{bmatrix}5\\6\end{bmatrix}$`, `$Av=\begin{bmatrix}17\\39\end{bmatrix}$` reads **“A times v equals the column vector seventeen, thirty-nine.”** Each output entry is a row's dot product with `v`: first row `$1(5)+2(6)=17$`; second row `$3(5)+4(6)=39$`. This is not component-by-component multiplication. `A` has shape `2 × 2`, `v` has shape `2 × 1`, and the result has shape `2 × 1`; the inner dimensions must agree.

If `$B=\begin{bmatrix}5&6\\7&8\end{bmatrix}$`, then `$AB=\begin{bmatrix}19&22\\43&50\end{bmatrix}$` is **“A times B.”** Each output entry combines one row of `A` with one column of `B`; for instance, the top-left entry is `$1(5)+2(7)=19$`. The entire operation composes two linear transformations. Order matters: in general `$AB\ne BA$`—read **“A B is not equal to B A.”**

- `$A^T$` is **“A transpose.”** Rows become columns: `$A^T=\begin{bmatrix}1&3\\2&4\end{bmatrix}$`. Transposing changes a `rows × columns` shape into `columns × rows`; it does not change the values themselves.
- `$A^{-1}$` is **“A inverse.”** It is a matrix that undoes multiplication by `A`: `$A^{-1}A=I$`, read **“A inverse times A equals the identity matrix.”** The identity `I` acts like `1` for matrix multiplication. Not every square matrix has an inverse. The course's example has `$A^{-1}=\begin{bmatrix}-2&1\\1.5&-0.5\end{bmatrix}$`.
- `$\det(A)$` is **“the determinant of A.”** For a `2 × 2` matrix `[[a,b],[c,d]]`, calculate `$ad-bc$`. For our `A`, `$1(4)-2(3)=-2$`. A zero determinant means a square matrix has collapsed some direction and cannot be inverted. The sign also encodes orientation; it is not a probability or model score.
- `$\operatorname{tr}(A)$` is **“the trace of A.”** Add its main-diagonal entries: `$1+4=5$`. This is a different summary from the determinant.

`$Av=\lambda v$` is **“A times v equals lambda times v.”** `λ` (lambda) is a scalar *eigenvalue*; nonzero `v` is an *eigenvector*. The complete statement says that applying `A` to this particular direction merely stretches, shrinks, or flips it without turning it into a new direction. With `$A=\operatorname{diag}(2,3)$` and `$v=(1,0)$`, `$Av=(2,0)=2v$`; therefore `v` is an eigenvector with eigenvalue `2`. This idea later helps explain principal component analysis (PCA), which finds directions that summarize variation in data.

### Sets and logical comparisons

For a set `$S=\{1,2,3\}$`, `$|S|=3$` is **“the cardinality of S is three”**—the number of distinct elements, not their numeric sum. Set bars and absolute-value bars look alike; context tells you whether you are counting a set or measuring a number.

Let `$U=\{1,2,3,4,5\}$` be the universe, `$A=\{1,2,3\}$`, and `$B=\{3,4\}$`. `$A\cup B=\{1,2,3,4\}$` reads **“A union B”**: elements in either set. `$A\cap B=\{3\}$` reads **“A intersection B”**: elements in both. `$A^c=\{4,5\}$` reads **“A complement”**: elements in the stated universe `U` but not in `A`. Without an explicit universe, “everything not in A” is ambiguous. In security operations, these can represent all alerts, alerts matching two rules, or events excluded by a rule.

### Logs and exponentials

`$\log_2(8)=3$` is **“log base two of eight equals three.”** It asks: *two to what power gives eight?* Since `$2^3=8$`, the answer is `3`. Logs turn repeated multiplication into addition and are useful when representing probabilities or information on a manageable scale.

`$\ln(e^2)=2$` is **“the natural log of e squared equals two.”** `e` is approximately `2.718`; `ln` means log base `e`, so it reverses raising `e` to a power. `$e^2\approx7.389$` is **“e squared is approximately seven point three eight nine.”** A base-two exponential `$2^x$` and a base-`e` exponential `$e^x$` grow at different rates, but both undo the corresponding logarithm. We will revisit `ln` when training objectives use log probabilities.

### Probability and spread

`$P(X\mid Y)$` is **“the probability of X given Y.”** The bar means *conditioned on*, not division. The complete idea is: after learning `Y` occurred, how likely is `X`? Importantly, `$P(\text{attack}\mid\text{alert})$` is not `$P(\text{alert}\mid\text{attack})$`. Suppose 1,000 events contain 10 attacks; a detector catches 9 attacks and raises 99 false alerts. Among the 108 alerts, only 9 are attacks, so the first probability is `$9/108\approx8.3\%$`. This **base-rate effect** matters when assessing real alert quality.

`$E[X]=\sum_i x_iP(X=x_i)$` is **“the expected value of X equals the sum over i of x sub i times the probability that X equals x sub i.”** For each possible outcome, multiply the outcome by its chance and add. If `$X=1$` with probability `0.25` and `$X=0$` with probability `0.75`, then `$E[X]=1(0.25)+0(0.75)=0.25$`. This is a probability-weighted average, not a promise that any single observation will equal `0.25`.

`$\operatorname{Var}(X)=E[(X-E[X])^2]$` is **“the variance of X equals the expected value of the squared difference between X and its expected value.”** Subtract the mean, square the distance, then average across outcomes. For the same `0/1` example, `$\operatorname{Var}(X)=0.75(0-0.25)^2+0.25(1-0.25)^2=0.1875$`. Variance is in *squared units*. `$\sigma(X)=\sqrt{\operatorname{Var}(X)}\approx0.433$` is **“sigma of X equals the square root of variance”**—the standard deviation, back in the original units.

`$\operatorname{Cov}(X,Y)=E[(X-E[X])(Y-E[Y])]$` is **“the covariance of X and Y equals the expected product of their deviations from their means.”** Positive covariance means they tend to move together; negative means they tend to move oppositely. `$\rho(X,Y)=\operatorname{Cov}(X,Y)/(\sigma(X)\sigma(Y))$` is **“rho of X and Y equals covariance divided by the product of their standard deviations.”** This standardized value lies from `-1` to `1` when both standard deviations are nonzero. For equally likely paired observations `(1,2)` and `(2,4)`, covariance is `0.5`, the standard deviations are `0.5` and `1`, and correlation is `$0.5/(0.5\times1)=1$`: a perfect positive linear relationship in this tiny example. Correlation does not by itself prove causation.

### Why these symbols matter to an AI red teamer

Norms quantify perturbations; matrix products describe neural-network layers; eigenvectors underpin PCA; conditional probabilities help interpret detector alerts; expectations and variance summarize uncertain outcomes. The symbols are compression for these ideas. Whenever a later formula looks intimidating, expand it into the same four steps used here: pronounce it, identify its parts, explain its purpose, and insert small numbers before looking at code.

HTB uses formula-like snippets for this refresher, not runnable Python assignments. We therefore keep this section as notes rather than adding a notebook that would imply the course introduced an implementation task.

## Section 3 of 24 — Supervised Learning Algorithms

This section defines the workflow that the following algorithm sections will instantiate. **Supervised** means that each example has a supplied answer. The learning task is to infer a reusable relationship, not to memorize a lookup table of training rows.

### A training record and the prediction loop

In the notation we will use to unpack HTB's prose, a training record is `$(x_i,y_i)$`—say **“the pair x sub i, y sub i.”** `i` identifies a row; `$x_i$` is that row's input features; `$y_i$` is its known label. A simple security row might have features such as message length, number of links, and sender reputation, with a label of phishing or benign. In data-engineering terms, features are columns available at decision time, while labels are the historical outcomes we want to learn to predict.

Write a model prediction as `$\hat y_i=f_\theta(x_i)$`: say **“y-hat sub i equals f sub theta of x sub i.”** The hat marks a *predicted* answer rather than the observed `$y_i$`; `$f$` is the model; `θ` (theta) represents its adjustable parameters. The complete sentence is: “With its current settings, the model turns row `i`'s features into a predicted label or value.” **Training** adjusts `θ`; **prediction** applies the learned settings to a new row. This notation is a learning aid for HTB's description, not a formula that HTB asks us to code in this section.

- **Classification** predicts a category: phishing versus benign, malware family A versus B, or cat versus dog.
- **Regression** predicts a numerical quantity: expected response time, transaction amount, or house price. A model's *score* for a category can be numeric even when the final task is classification; the task type depends on the desired output.

### Training, evaluation, and generalization

The basic flow is **collect labeled examples → separate training from evaluation data → fit parameters on training examples → evaluate on unseen examples → deploy and monitor**. The *training set* teaches the model. A *validation set* helps select settings and compare versions. A final *test set* estimates performance after those choices are fixed. Repeatedly looking at and tuning against the test set turns it into another validation set, giving an overoptimistic estimate. In time-ordered security data, a chronological split may be more honest than a random split: future campaigns must not leak into the past.

**Generalization** means performing well on new cases, not merely on records already seen. **Overfitting** is learning training-specific noise or quirks: training performance is excellent but new-case performance drops. **Underfitting** is failing to capture an important pattern at all: both training and new-case performance are poor. A model that memorizes attacker IPs from last month's logs might overfit and miss a new attacker using new infrastructure. A model that only checks message length might underfit a phishing problem.

**Cross-validation** repeatedly rotates which subset is held out: in five-fold cross-validation, train on four folds and evaluate on the fifth, then rotate through all five. This gives multiple estimates instead of trusting one arbitrary split. It does not cure leakage: near-duplicate samples, events from the same incident, or later events can still appear on both sides unless groups and time are respected.

HTB distinguishes *prediction* from the broader word *inference*. In applied ML, “model inference” often simply means running a trained model to obtain a prediction; in statistics, “inference” can mean estimating or explaining underlying relationships. Ask which sense a speaker means. Explaining a model's correlation is not the same as proving a causal relationship.

### Security evaluation: why accuracy alone can deceive

Consider 1,000 events: 10 are real attacks and 990 are benign. A detector catches 8 attacks, misses 2, and falsely alerts on 12 benign events. Read the four counts as follows:

| Actual versus predicted | Predicted attack | Predicted benign |
| --- | ---: | ---: |
| Actual attack | 8 **true positives** (TP) | 2 **false negatives** (FN) |
| Actual benign | 12 **false positives** (FP) | 978 **true negatives** (TN) |

- `$\text{Accuracy}=(TP+TN)/(TP+TN+FP+FN)$` is **“accuracy equals true positives plus true negatives, divided by all cases.”** It measures the fraction of all correct decisions: `$(8+978)/1000=98.6\%$`. That sounds excellent, yet many alerts are wrong. A detector that always said “benign” would achieve `99%` accuracy here while catching **zero** attacks.
- `$\text{Precision}=TP/(TP+FP)$` is **“precision equals true positives divided by all predicted positives.”** Of the events this model called attacks, how many were really attacks? `$8/(8+12)=40\%$`. Six out of ten alerts waste an analyst's time.
- `$\text{Recall}=TP/(TP+FN)$` is **“recall equals true positives divided by all actual positives.”** Of the real attacks, how many were caught? `$8/(8+2)=80\%$`. The remaining 20% slipped through.
- `$F_1=2PR/(P+R)$` is **“F-one equals two times precision times recall, divided by precision plus recall.”** Here `P` means precision and `R` means recall; the result is `$2(0.4)(0.8)/(0.4+0.8)\approx0.533$`. It is a *harmonic* mean: a low value in either ingredient pulls the score down. If both `P` and `R` are zero, define F1 as zero in this simple setting.

These formulas are added to make HTB's metric names concrete. They describe different operational questions, so choose metrics based on the cost of missed attacks versus false alerts. A single number never replaces examining the confusion matrix, class frequency, and the decision threshold.

### Regularization: discouraging an over-complex fit

HTB introduces **regularization** as adding a complexity penalty to the training loss. A generic objective can be written `$L_{\text{total}}(\theta)=L_{\text{data}}(\theta)+\lambda R(\theta)$`: say **“L-total of theta equals L-data of theta plus lambda times R of theta.”** `$L_{\text{data}}$` measures prediction mistakes, `$R$` measures model complexity, and `λ` (lambda) controls how strongly to penalize complexity. The complete idea is “fit the examples, but pay a price for a model whose parameters become too large or complicated.” This equation spells out HTB's prose; the course does not implement it yet.

For coefficient vector `$w=(w_1,\ldots,w_n)$`, an **L1 penalty** is `$R_1(w)=\sum_i|w_i|$`: say **“R-one of w equals the sum over i of the absolute value of w sub i.”** With `$w=(2,-3)$`, the penalty is `$|2|+|-3|=5$`. An **L2-squared penalty** is `$R_2(w)=\sum_iw_i^2$`: say **“R-two of w equals the sum over i of w sub i squared.”** For the same vector it is `$2^2+(-3)^2=13$`. L1 can encourage some coefficients to become exactly zero; L2-squared discourages large coefficients smoothly. These are penalties on **model parameters**. Later adversarial-attack norms often measure an **input perturbation** instead—same mathematical shapes, different objects and purposes.

### Security implications beyond the course's introductory list

- **Label quality:** if “benign” labels actually include undiscovered attacks, the training target is noisy. An attacker deliberately corrupting examples or labels is attempting **data poisoning**.
- **Feature availability:** a feature computed using facts discovered *after* an incident cannot be used by a real-time detector. Including it during training is **data leakage** and can make the model appear far better than it will be in production.
- **Distribution shift:** attackers change tooling and infrastructure, and legitimate traffic also changes. Monitor performance after deployment; yesterday's cross-validation score is not a permanent guarantee.
- **Evasion:** an adversary can modify an input at prediction time to cross the model's decision boundary while preserving the attack's function. This is distinct from poisoning training data.

This section is conceptual. We keep the examples in prose and arithmetic, then reserve annotated training code for the specific algorithms that HTB introduces next.

## Section 4 of 24 — Linear Regression

Linear regression is the first concrete model in the supervised-learning sequence. The label is a **number**, not a category. HTB uses house price and size as an intuition; our small companion notebook uses a synthetic request-size/latency example. That example demonstrates the arithmetic, not a realistic performance model or a security detector.

### One input: a line with two adjustable settings

HTB writes `$y=mx+c$`. Say **“y equals m times x plus c.”** `x` is the input feature, `m` is the *slope*, and `c` is the *intercept*. In prediction notation, it is clearer to write `$\hat y=mx+c$`—**“y-hat equals m times x plus c”**—because the actual observed `y` may differ from the line's prediction. The whole equation says: start with a baseline prediction `c`, then add `m` units for each unit of `x`. For `$m=2$` and `$c=1$`, at `$x=3$` the prediction is `$\hat y=2(3)+1=7$`. If `m` were negative, predictions would fall as `x` rises. The intercept may be mathematically necessary even when `x=0` is outside the meaningful data range; do not interpret it as a real-world zero-input measurement without checking that range.

This is a model assumption, not a law of nature. A straight line assumes a one-unit increase in `x` changes the prediction by the same amount throughout the modeled range. Security measurements often violate that assumption: traffic can saturate a service, so latency may bend upward at high load. Extrapolating far beyond training values is especially risky.

### Several inputs: contributions add together

HTB writes `$y=b_0+b_1x_1+b_2x_2+\cdots+b_nx_n$`. Say **“y equals b sub zero plus b sub one times x sub one plus b sub two times x sub two, and so on through b sub n times x sub n.”** `x_i` is feature `i`; `b_i` is that feature's coefficient; `b_0` is the intercept. For a prediction, read the left side as `y-hat`. The whole equation says to add a baseline to each feature's weighted contribution. For `$\hat y=2+3x_1-4x_2$`, inputs `$x_1=2$` and `$x_2=1$` produce `$2+3(2)-4(1)=4$`. Here `b_2=-4` means that increasing `x_2` by one changes the prediction by `-4` **if `x_1` is held fixed**. It does not prove that changing `x_2` causes the outcome to change: correlated features and omitted variables complicate that interpretation.

There is still no multiplication *between* features in HTB's equation. A line can have many input columns and still be “linear” in its coefficients. More features do not automatically make it more accurate on new data; they can also increase leakage and overfitting risk.

### Ordinary least squares: choose the best-fitting settings

For row `i`, write `$r_i=y_i-\hat y_i$`: say **“r sub i equals y sub i minus y-hat sub i.”** `r_i` is the *residual*, the signed gap from the observed value to the prediction. A positive residual means the model predicted too low; a negative one means it predicted too high. For a true value `5` and prediction `4`, the residual is `1`.

The residual sum of squares is `$RSS(m,c)=\sum_{i=1}^{N}(y_i-(mx_i+c))^2$`. Say **“R-S-S of m and c equals the sum from i equals one to N of y sub i minus open-parenthesis m times x sub i plus c close-parenthesis, all squared.”** `N` is the number of training rows; the inner subtraction is that row's residual; the square makes it nonnegative and makes a large miss count disproportionately; the sum produces one score for a candidate line. The whole equation says: *add up the squared prediction errors across every training row*. OLS chooses `$\operatorname*{arg\,min}_{m,c}RSS(m,c)$`—say **“the values of m and c that minimize R-S-S.”** `arg min` returns the **settings** that make the score smallest, not the smallest score itself.

Try three observations: `$(x,y)=(1,2),(2,3),(3,5)$`. The candidate line `$\hat y=x+1$` predicts `2, 3, 4`, so its residuals are `0, 0, 1` and `RSS=0^2+0^2+1^2=1`. OLS instead finds `$m=1.5$` and `$c=1/3$`. Its predictions are `$11/6,10/3,29/6$`; residuals are `$1/6,-1/3,1/6$`; and `RSS=1/36+1/9+1/36=1/6`, smaller than `1`. The line does not pass through every observation, because the goal is to minimize the **total** squared error.

For many features, the same idea applies: adjust all the `b` coefficients to minimize squared residuals. The companion notebook uses `numpy.linalg.lstsq`, a standard numerical least-squares solver, for this tiny example. That solver is a practical implementation choice, **not** code supplied by HTB in this section. We avoid teaching an explicit matrix inverse as the default algorithm; numerical solvers are typically safer and more stable.

### Assumptions and what happens when they fail

HTB lists four assumptions. **Linearity** means the conditional mean of the target is adequately represented by the additive straight-line formula. **Independence** means rows' errors are not systematically tied together; repeated measurements from one host or incident may violate it. **Homoscedasticity**, pronounced **“ho-mo-skeh-DAS-tiss-ity,”** means residual spread is roughly constant across the range of fitted values. If high-load measurements have much wider errors, uncertainty is not constant. **Normality** refers to residual/error distribution and is especially relevant to classical confidence intervals and significance tests; an imperfect bell shape does not automatically make point predictions useless. These are modeling assumptions to *check*, not properties guaranteed by calling an OLS function.

For a security use case, a small training RSS does not prove the model works on a new campaign or system. Inspect residuals, evaluate on held-out data, and check that input features are available at prediction time. A malicious or simply unusual high-leverage training point can pull an OLS line substantially because squaring emphasizes large residuals. That is a robustness concern beyond HTB's introductory explanation, not an attack we implement here.
