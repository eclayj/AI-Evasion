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
