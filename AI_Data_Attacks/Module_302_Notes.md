# AI Data Attacks — COAE Study Notes

Source: HTB Academy, [AI Data Attacks](https://academy.hackthebox.com/app/module/302). These notes follow the module in order and separate HTB's lesson points from additional security context. A notebook is added only when a section contains a useful runnable example or exercise; this opening section is a pipeline overview, so no notebook is needed yet.

## Section 1 of 25 — Introduction to AI Data

### The core idea

An AI model learns patterns from the data and processing it receives. If an attacker can corrupt the examples, alter the transformation code, replace the saved model, or poison feedback used for retraining, the model may carry that influence into its predictions. The security question is therefore broader than “is the model code secure?” We also ask who can supply, change, read, process, and deploy the data and model artifacts.

HTB's high-level pipeline is:

**collect → store → process and transform → model → deploy → monitor and maintain → feed new data into retraining**

The course diagram compresses some of these steps into “utilize,” meaning the prepared data is used either to train a model or generate predictions. The expanded steps help locate where an attacker could interfere.

### Walk through the pipeline

1. **Collect.** Applications, databases, sensors, websites, or third parties provide raw data. Examples include JSON application logs, PostgreSQL records, MQTT sensor messages, and JPEG images. Each source is a trust boundary: the pipeline must not assume that a value is authentic just because it arrived through an expected format or channel.
2. **Store.** Data may live in relational databases, NoSQL databases, data lakes, or time-series databases. Trained models are stored too, often as files such as `.pkl`, `.pt`, or ONNX. Protecting only the original dataset is insufficient if someone can replace the model file that production later loads.
3. **Process and transform.** Cleaning, scaling, feature extraction, and other code turn raw records into model inputs. If an attacker changes a preprocessing job, the upstream raw dataset may still look untouched while the model receives altered features. Validation should cover both input data and the code that transforms it.
4. **Model.** Analysts explore data, choose algorithms, tune settings, and evaluate the result. The model can learn a malicious pattern if that pattern was planted upstream; ordinary model training does not determine whether a pattern is legitimate.
5. **Deploy.** The model is loaded into an API, serverless function, container, or edge device. Deployment turns a stored artifact into a live component. Integrity checks, trusted loading paths, and access controls matter here.
6. **Monitor and maintain.** Teams watch service health and model behavior, then may retrain using new data. A feedback loop can become an attack route if user-controlled inputs or predictions are automatically treated as trustworthy training examples.

### Security vocabulary in this section

- **Trust boundary:** a point where data, authority, or control crosses between components or parties. Crossing a boundary is a reason to validate provenance, format, permissions, and integrity; it is not proof that the data is malicious.
- **Data pipeline:** the connected systems and jobs that collect, store, transform, and use data. A pipeline includes infrastructure and scripts, not just the final CSV.
- **Data lake:** large-scale storage for data in varied formats, often before it has been shaped into a specific table or model input.
- **Feature engineering:** code that derives model inputs from raw fields—for example, extracting hour-of-day from a timestamp. A compromised feature extractor can alter what the model learns without modifying the original records.
- **Serialized model / model artifact:** a model saved in a portable file for later loading. `.pkl` commonly refers to Python pickle serialization; `.pt` is commonly used for PyTorch artifacts; ONNX is a format for exchanging models across tools. File extensions alone do not prove safety or authenticity. Pickle-style loading of untrusted files is especially risky because deserialization can execute code; other formats still need provenance and integrity checks.
- **Data poisoning:** deliberately influencing training data or its labels so the resulting model learns attacker-chosen behavior. This module later distinguishes poisoning routes and effects.
- **Online poisoning:** poisoning that enters through data collected during operation—such as feedback or user interactions—and influences a later retraining cycle. “Online” here means during the system's deployed lifecycle; it does not necessarily mean the model updates after every single request.
- **Data drift:** a change in the distribution of incoming data over time. Drift may be natural, adversarial, or both; observing drift is a signal to investigate, not a verdict that an attack happened.

### Two examples and their lessons

HTB contrasts an e-commerce recommender, with streamed activity and recurring retraining, against a healthcare model whose data and updates are more tightly controlled. Their impact and safeguards differ, but both depend on trusted collection, storage, processing, model files, and feedback. Security review should follow the complete lifecycle and identify which component has write access to the next one.

The examples mention tools such as Kafka (a system for streaming event data), Spark (distributed data processing), S3-style object storage, Docker/Kubernetes (container packaging and orchestration), and REST APIs (a common request/response interface). The security concept does not depend on these specific vendors: changing a tool does not remove the data-integrity boundary.

### COAE quick recall

When a model behaves unexpectedly, trace the lifecycle from source to prediction: **Who supplied the data? What transformed it? Where is the model artifact stored? Which identity can replace or load it? Does runtime feedback enter retraining?** Check integrity and access controls at each handoff. Clean-looking source data does not rule out compromised transformation code, and a clean training process does not rule out a swapped deployment artifact. Treat drift as a clue to investigate, not proof of poisoning.

### What this section does and does not establish

HTB introduces the attack surface and previews later topics: training-set corruption, feature manipulation, Trojans, model artifacts, and tensor steganography. This section does not provide a specific exploit procedure or a runnable code block; those techniques belong to their later sections. The practical mitigations here are lifecycle visibility, least-privilege access, provenance and integrity checks, and controlled retraining. These are general security connections to the overview, not a claim that one control alone solves every later attack.

## Section 2 of 25 — AI Data Attacks

### Place the attack in the lifecycle

This section contrasts three broad goals:

- **Data attacks** alter the information or artifact that shapes the model, so the effect may persist in the trained or deployed system.
- **Evasion attacks** alter an input at inference time to make a deployed model misclassify it. The attacker need not change the training set.
- **Privacy attacks** try to infer sensitive information from the model or its surrounding system. Their target is confidentiality, not necessarily prediction integrity.

An incident can combine categories. For example, a poisoned dataset may create a hidden backdoor, and a privacy weakness may expose records used during training. Classify the behavior you can verify, then describe the chain rather than forcing a multi-stage attack into only one label.

### Attack locations and what changes

| Pipeline location | What the attacker changes | Example from the lesson | Useful distinction |
| --- | --- | --- | --- |
| Collection | Submitted examples or labels | Fake product reviews enter through the ordinary review form | The normal format or API does not make a contribution trustworthy |
| Stored dataset | Records after collection | Labels or features are edited in a data lake | Ingestion validation can be bypassed by later write access |
| Processing code | Cleaning, transformation, or feature-extraction logic | A Spark job flips sentiment labels while raw reviews remain unchanged | The raw dataset may pass review even though model inputs are corrupted |
| Training/modeling | The examples the training job receives | A model learns the poisoned labels or trigger pattern | Training normally optimizes for the provided objective; it does not establish that the data is honest |
| Model storage or deployment path | The model artifact or the file fetched for production | A trojaned or substituted artifact is loaded by the serving system | Deployment-path access can matter even without write access to the original data store |
| Feedback and retraining | New user interactions or feedback used in later training | Gradually biased click or rating data enters recurring retraining | Benign data drift and malicious manipulation can look similar; drift is a signal to investigate, not proof of attack |

### Distinguish poisoning, a backdoor, and unsafe model loading

**Data poisoning** is the broader act of manipulating data used to train, tune, or update a model. **Label flipping** changes the answer associated with a sample—for example, training rows for “positive” reviews are marked “negative.” A **feature attack** changes the input representation, such as an image's pixel values or features extracted from text. Those terms describe *what is altered*.

A **backdoor** describes a hidden behavior learned or placed into a model: ordinary inputs receive expected outputs, but an input containing a trigger causes an attacker-chosen result. A poison attack can plant the trigger during training, but a backdoor can also be introduced by other model-tampering routes. The trigger is a condition for the behavior, not the attack's entry point.

**Model artifact tampering** changes or replaces the saved model. **Unsafe deserialization** is a separate loading risk: certain formats and APIs can execute code while reconstructing an object. HTB's example calls out Python pickle loading. Do not generalize this into “every `.pt` or ONNX file executes code.” The actual risk depends on serialization format, loader behavior, and how it is configured. Verify provenance and integrity, and use safer loading modes where supported.

HTB also uses the phrase **model steganography** for hiding code in a model file. Keep that course term distinct from the general security conclusion: a hidden payload may be inert unless a vulnerable loader or another execution path activates it. A backdoor that changes predictions and a payload that achieves server-side code execution are different impacts, even if they coexist in one artifact.

### Framework mapping: preserve the version labels

HTB's section maps data poisoning to **OWASP LLM03: Training Data Poisoning** and supply-chain risks to **LLM05: Supply Chain Vulnerabilities**. These names and numbers match OWASP's **2023–24** LLM Top 10. OWASP's **2025** list renumbered these topics: **LLM03:2025 is Supply Chain**, **LLM04:2025 is Data and Model Poisoning**, and **LLM05:2025 is Improper Output Handling**. For an exam question anchored to this HTB page, learn the page's mapping; when discussing the current list, state the year and use the 2025 labels. [OWASP's 2023–24 LLM03](https://genai.owasp.org/llmrisk2023-24/llm03-training-data-poisoning/) and [LLM05](https://genai.owasp.org/llmrisk2023-24/llm05-supply-chain-vulnerabilities/) preserve the course's labels; see also [OWASP LLM03:2025 Supply Chain](https://genai.owasp.org/llmrisk/llm032025-supply-chain/) and [LLM04:2025 Data and Model Poisoning](https://genai.owasp.org/llmrisk/llm042025-data-and-model-poisoning/). Crosswalk checked 2026-11-14.

The lesson also connects the lifecycle to Google's **SAIF** (Secure AI Framework): secure design, data protection, supply-chain assurance, security testing, secure deployment, and monitoring/response. Think of OWASP as a way to name vulnerability categories and SAIF as a way to organize security work across the AI lifecycle. Neither label replaces tracing the actual source, transformation, artifact, permission, and effect in a specific incident.

### COAE quick recall

For a scenario, ask **what is targeted, when does it happen, and what property is harmed?** Training examples or labels point to poisoning (integrity); a trigger that changes predictions points to a backdoor (integrity); a changed input at prediction time points to evasion; extraction of learned private information points to a privacy attack (confidentiality); and a compromised source/model/dependency/deployment artifact points to supply-chain risk. Then locate the pipeline stage and identify the security boundary that failed. One attack can traverse several stages.

**Version check:** answer using the OWASP version named in the question. HTB's 2023–24 mapping and OWASP's 2025 numbering differ, so do not quote the old LLM03/LLM05 numbers as if they were the 2025 list.

### Notebook decision

This is a conceptual threat-mapping section and supplies no runnable code. The table above serves as the review aid; adding a notebook would repeat it without creating a useful exercise.

## Section 3 of 25 — Label Flipping

### The attack in plain language

Training examples have two parts: the **features** (the information describing an example) and the **label** (the answer the model is supposed to learn). Label flipping changes selected training labels while leaving their features alone. The lesson's examples are changing an image label from “cat” to “dog,” or an email label from “spam” to “not spam.” A model trained on contradictory examples may learn a weaker or misleading relationship between inputs and classes.

For a binary class encoded as 0 or 1, a compact teaching expression is `y_i' = 1 - y_i`. Read it **“y sub i prime equals one minus y sub i.”** Here `y_i` is example i's original label, and `y_i'` (“y sub i prime”) is its changed label. If the original label is 0, then 1 − 0 = 1; if it is 1, then 1 − 1 = 0. This formula applies only to binary 0/1 labels. It is a teaching notation for the idea, not a formula shown by HTB in this section.

The example vector is `x_i = (x_i1, x_i2)`. Say it **“x sub i equals the pair x sub i one, x sub i two.”** `i` identifies a row/example; `x_i1` and `x_i2` are that row's first and second features. The attack leaves the pair `x_i` unchanged and changes only `y_i`. This contrasts with a feature attack, which changes the input values themselves.

### What HTB's code sets up

The page imports NumPy, Matplotlib, scikit-learn, and Seaborn; sets plot colors; and fixes a random seed at 1337. It uses `make_blobs` to create 1,000 artificial two-feature examples around centers `(0, 5)` and `(5, 0)`, with a cluster standard deviation of 1.25. The two clusters stand in for negative and positive sentiment. They are not real reviews, and the feature dimensions are not words or a real sentiment representation.

The data is split into 70% training examples and 30% test examples with a fixed random state. HTB's visible code then plots the clean training split so the starting class groups can be seen. The code shown in this subsection builds and visualizes the baseline data; it does not yet show the label-changing operation. The accompanying notebook reproduces those current-section building blocks and annotates them for study. It keeps the labels of the held-out test rows untouched, ready for a later comparison.

### Why the held-out set matters

If the goal is to see whether poisoned training labels harmed performance, evaluate on test examples whose labels remain correct. Flipping test labels too would corrupt the measuring stick: a model could appear better or worse simply because the answers used for scoring were changed. That clean-test rule is a sound evaluation practice connected to the exercise; the current HTB code shown here only creates the split and clean visualization.

### COAE quick recall

Given a suspicious training set, compare **features** with **labels**. If the inputs are unchanged but some answers are deliberately wrong, classify it as label flipping (a data-poisoning attack on integrity). A broad decrease in accuracy, precision, or recall is a possible consequence, not a guarantee: impact depends on which labels are changed, how many, the data distribution, and the model. Keep the test labels clean when measuring the impact.

Notebook: [Module_302_Label_Flipping_Study.ipynb](Module_302_Label_Flipping_Study.ipynb). It currently covers the synthetic clean dataset, split, and plot from this section; it does not implement future attack steps.

## Section 4 of 25 — Baseline Logistic Regression Model

### Why establish a baseline?

A **baseline** is the reference result from the original, clean training data. Later, when the training labels are deliberately corrupted, we can compare the changed model against this clean reference. The comparison is meaningful only if the evaluation examples and their correct labels remain held out and unchanged. In the notebook, the model learns from `X_train, y_train`; we score it on `X_test, y_test`.

### Read the model's score from the features

The lesson represents example *i* by `x_i = (x_i1, x_i2)`, read **“x sub i equals the pair x sub i one and x sub i two.”** The model first calculates

`z_i = w^T x_i + b = w_1 x_i1 + w_2 x_i2 + b`.

Read this **“z sub i equals w transpose times x sub i, plus b,”** or in expanded form **“w one times x sub i one, plus w two times x sub i two, plus b.”** The superscript `T` means transpose. In this expression it tells us to take a dot product: multiply each feature by its matching weight, then add the products. The weights `w_1` and `w_2` describe how the two features influence the score; `b` is the bias, which shifts the boundary.

For a tiny numerical example, let `w = (0.8, -0.6)`, `x_i = (2, 1)`, and `b = -0.3`. Then `z_i = 0.8×2 + (-0.6)×1 - 0.3 = 0.7`. The first feature pushes toward class 1, the second pushes away from it, and the bias shifts the total. The model's score is positive, so it leans toward class 1.

The score is a **logit** or **log-odds**:

`z_i = log( P(y_i = 1 | x_i) / (1 - P(y_i = 1 | x_i)) )`.

Say it **“z sub i equals the log of the probability that y sub i is one given x sub i, divided by one minus that probability.”** The vertical bar means “given.” The fraction is the odds of class 1 versus class 0; the logarithm turns those odds into a score that can be any real number. For example, probability `0.75` means odds `0.75 / 0.25 = 3`, and its natural log is about `1.099`, a positive score favoring class 1.

The sigmoid converts that score back to a probability:

`p_i = σ(z_i) = 1 / (1 + e^(-z_i))`.

Read this **“p sub i equals sigma of z sub i, equals one over one plus e to the negative z sub i.”** `σ` is the sigmoid function and `e` is Euler's number. If `z_i = 0`, then `p_i = 0.5`; if the score is positive, the probability is above `0.5`; if negative, it is below `0.5`. For the example score `0.7`, sigmoid gives approximately `0.668`, so the model assigns about a 66.8% probability to class 1.

### How training chooses weights

HTB gives the binary cross-entropy (BCE), also called log-loss:

`L(w,b) = -(1/N) Σ_(i=1)^N [ y_i log(p_i) + (1-y_i) log(1-p_i) ]`.

Read it **“L of w and b equals negative one over N times the sum, for i from one to N, of y sub i log p sub i plus one minus y sub i times log of one minus p sub i.”** `N` is the number of training examples; `Σ` means add the bracketed loss for each example; `y_i` is the correct 0/1 label; and `p_i` is the predicted probability of class 1. The expression chooses the appropriate probability: if `y_i=1`, the first term is `log(p_i)`; if `y_i=0`, the second term is `log(1-p_i)`. The negative sign makes confident correct predictions low-loss and confident wrong predictions high-loss.

For a class-1 example, predicting `p_i=0.9` gives loss `-log(0.9) ≈ 0.105`; predicting `p_i=0.1` gives `-log(0.1) ≈ 2.303`. Training uses an optimizer (the course names gradient descent) to adjust `w` and `b` to reduce the average loss across the training examples. Scikit-learn's `.fit()` performs that optimization for this logistic regression model.

### The decision boundary and evaluation

The prediction rule is class 1 when `p ≥ 0.5`, read **“p is greater than or equal to one half,”** and class 0 otherwise. Sigmoid equals `0.5` exactly when `z=0`, so the boundary is

`w^T x + b = 0`, or in two dimensions `w_1 x_1 + w_2 x_2 + b = 0`.

This is a straight line separating the two predicted regions. For example, if `w_1=2`, `w_2=-1`, and `b=-3`, the boundary equation is `2x_1 - x_2 - 3 = 0`; solving for `x_2` gives `x_2 = 2x_1 - 3`. A point on the line has probability 0.5; points on either side receive different predicted classes.

The lesson fits `LogisticRegression` on the clean training split, predicts the test split, and computes `accuracy = correct predictions / all test examples`. Its displayed result is `0.9933` (99.33%) for this easy, well-separated synthetic dataset. This high value establishes the reference for the exercise; it does not predict performance on real reviews. Accuracy can also hide class-specific failures in imbalanced datasets, so interpret it with the data distribution and other appropriate metrics in a real evaluation.

The decision-boundary plot evaluates the model over a grid of feature coordinates and colors each grid location by its predicted class. It visualizes the fitted line and the two regions; it is not a second training procedure. The notebook uses a slightly coarser mesh than HTB's `0.02` step (`0.05`) so the teaching plot uses fewer temporary points while remaining clear.

### COAE quick recall

- **Logistic regression:** a linear score plus sigmoid yields a binary-class probability.
- **Logit / log-odds:** `z = w^T x + b`; positive favors class 1, negative favors class 0.
- **Sigmoid:** maps a real-valued score to `(0,1)`; `z=0` maps to `0.5`.
- **BCE / log-loss:** rewards assigning high probability to the correct label and penalizes confident mistakes.
- **Decision boundary:** where `p=0.5`, equivalently `z=0`; for two features, it is a line.
- **Baseline integrity:** train on clean training data; compare on the same untouched, correctly labeled test set after poisoning.

**Exam check:** if a question asks why establish a baseline before poisoning, answer that it provides a clean reference for measuring the attack's effect. If asked whether the lesson's 99.33% should be expected in production, answer no: it is a result on a small, artificial, easily separated dataset.

### Notebook update

At the end of this subsection, the notebook contains the clean logistic-regression fit, held-out accuracy, and decision-boundary visualization. Section 5 then adds the label-flipping operation.


## Section 5 of 25 — The Label Flipping Attack

### What changes, and what stays fixed?

The attack targets the **training labels**. For each selected example, the feature vector (the information describing it) stays fixed while its answer is changed. In this sentiment exercise, the labels are binary: 0 means negative and 1 means positive. The attack is untargeted at the data-label level: every selected 0 becomes 1 and every selected 1 becomes 0; it does not select one desired destination class.

HTB's function takes the original label array `y` and a `poison_percentage`. It computes `n_to_flip = int(n_samples * poison_percentage)`, chooses that many distinct indices, copies the label array, changes only the copied entries, and returns both the new labels and selected indices. For 700 training examples at 10%, the count is `int(700 × 0.10) = 70`. The integer conversion drops any fractional remainder; very small percentages can therefore select zero rows.

### Pronounce the notation and read the loss change

- `y_i` — say **“why sub i”**: the label supplied for example *i*.
- `y_i′` — say **“why sub i prime”**: the changed version of that label. “Prime” here means modified, not exponentiation.
- `x_i` — say **“ex sub i”**: example *i*'s feature vector. It stays fixed in this attack.
- `p_i` — say **“pee sub i”**: the model's predicted probability of class 1 for example *i*.
- `−log(p_i)` — say **“negative log of pee sub i.”** For a class-1 label, it is the example's binary cross-entropy loss.
- `−log(1 − p_i)` — say **“negative log of one minus pee sub i.”** For a class-0 label, it is that example's loss.

The one-example binary cross-entropy from Section 4 is

`−[y_i log(p_i) + (1 − y_i) log(1 − p_i)]`.

Read it aloud: **“negative, open bracket, y sub i times log of p sub i, plus one minus y sub i times log of one minus p sub i, close bracket.”** If `y_i=1`, the first part is selected and the loss is `−log(p_i)`. If `y_i=0`, the second part is selected and the loss is `−log(1−p_i)`. In each case, a high predicted probability for the supplied class means a smaller loss.

Example: a review truly labeled 0 has predicted positive probability `p_i=0.1`. With the correct label, loss is `−log(1−0.1)=−log(0.9)≈0.105`. After changing its label to `y_i′=1`, loss for the same model prediction is `−log(0.1)≈2.303`. The poisoned label turns a confident, correct prediction into a large training error. During fitting, that error influences the learned weights and bias, potentially moving the decision boundary away from the one supported by the true data.

The general binary-flip shorthand is `y_i′ = 1 − y_i`, read **“y sub i prime equals one minus y sub i.”** If the old label is 0, the new label is `1−0=1`; if it is 1, the new label is `1−1=0`. This shortcut is only appropriate for 0/1 labels. HTB's implementation spells the choice out with `np.where`, which is easier to read when first learning the operation.

### How the code selects and changes rows

A seeded random-number generator makes the selection repeatable: running the same data, fraction, and seed selects the same indices. `choice(..., replace=False)` samples without replacement, so no row gets counted twice. Copying `y` before assignment protects the clean training labels. `np.where(old_label == 0, 1, 0)` expresses the two cases explicitly. Returning `flipped_indices` provides traceability: we can verify the number of changed rows and mark them in the plot.

The study notebook accepts `seed=SEED` as an optional function parameter, whereas the HTB page reads a global `SEED` inside the function. The resulting selection is the same, but the explicit parameter makes the dependency visible and the function easier to reuse. The notebook also checks that inputs are binary and that the requested percentage lies between 0 and 1; these are defensive guardrails around the lesson's core operation.

### What the visualization tells us—and does not tell us

The two-dimensional toy plot keeps every point at its original feature coordinates. The point's fill color reflects its current (possibly poisoned) label, and a red `X` marks a changed label. This makes it easy to see that the rows moved in **label space**, not feature space. It does not show training impact yet: Section 5 defines and visualizes the corruption, but does not fit a poisoned model or report its changed accuracy. That comparison belongs to later course material.

The notebook uses a 10% poison rate only as an illustrative setting. It preserves the clean training labels and untouched test labels, creates a separate poisoned training-label copy, and prints the actual number selected. A clean test set is important because it gives a reliable reference for later measuring how training-time poisoning changed model behavior.

### COAE quick recall

- **Attack family:** data poisoning, because the attacker alters data consumed during training.
- **Altered field:** labels (`y`); features (`X`) remain unchanged.
- **Security property:** training-data integrity. The model is optimized against answers that have been deliberately falsified.
- **Poison rate:** the fraction of training rows selected. It is not the same as measured attack impact.
- **Reproducibility:** a fixed seed enables the same random sample to be selected again; sampling without replacement avoids duplicate indices.
- **Evaluation hygiene:** keep the held-out test data and its labels clean.
- **Do not overclaim:** label flipping can move a learned decision boundary or reduce performance, but the effect depends on data, model, selected rows, and poison rate. This section's plot demonstrates label corruption, not a measured performance drop.

**Exam check:** if input features are unchanged but some training answers are deliberately reversed, identify label-flipping data poisoning. If asked why not alter test labels too, explain that the clean test labels are the trusted answer key for measuring the attack's effect.

At the end of Section 5, the notebook adds an annotated flip function and a visualization while preserving clean labels. Section 6 below extends it with the poisoned-model evaluation.


## Section 6 of 25 — Evaluating the Label Flipping Attack

### What is compared?

Section 5 created a poisoned copy of the training labels. This section uses those labels to fit new logistic-regression models, then checks each model against the **same untouched test features and correct test labels**. The clean baseline gives the reference. The experiment tries 10%, 20%, 30%, 40%, and 50% flipped training labels, with 0% representing the baseline.

For a poisoning rate `q`, the number changed is `int(N_train × q)`. Read `q` as **“cue”** and `N_train` as **“N train.”** It is a fraction between zero and one: `q=0.10` means 10%. With 700 training examples, `int(700 × 0.10)=70` labels are flipped. `q` describes how much training data was changed; it is not itself a measure of success or damage.

### Accuracy notation, spoken aloud

Accuracy is the fraction of test predictions that match the correct labels:

`accuracy(q) = (1 / N_test) Σ_(i=1)^N_test 1[model_q(x_i) = y_i]`.

Read this aloud: **“accuracy at q equals one over N test times the sum, for i from one to N test, of the indicator that model q predicts x sub i equal to y sub i.”**

- `N_test` is the number of held-out examples.
- `Σ` (capital sigma) means add one value for every test example.
- `model_q` means the model trained using poisoning fraction q.
- `x_i` is the feature vector for test row i; `y_i` is its correct label.
- `1[condition]` is an **indicator**: it equals 1 if the condition is true and 0 otherwise.

The whole sum counts correct predictions. Dividing by `N_test` turns that count into a proportion. If 297 of 300 predictions are correct, accuracy is `297/300 = 0.99`, or 99%. The notebook uses scikit-learn's `accuracy_score(y_test, predictions)` to calculate this fraction.

The feature rows used for poisoned training stay `X_train`; only their training labels differ. Crucially, `y_test` remains the true answer key. Poisoning the test labels too would make the reported score untrustworthy because we'd be grading predictions against deliberately false answers.

### Why a boundary can move before accuracy drops

The decision boundary from Section 4 is the set of feature points where `w^T x + b = 0`, read **“w transpose times x plus b equals zero.”** It is also where the predicted class-1 probability is `0.5`. Poisoned training examples alter the loss the model tries to minimize, so the learned weights and bias may change. That moves the line.

But accuracy is measured only at the held-out test points. If the line moves without crossing any of those points, every test prediction remains the same, so accuracy stays the same. HTB reports this at 10%: both clean and poisoned models have 99.33% accuracy on the synthetic test set, although the boundary has shifted slightly. There is no contradiction: one measure describes a geometric model change; the other counts changed test decisions.

As more labels are corrupted, the model is pulled by more incorrect examples and its boundary can move further. HTB's clear, separated toy clusters retain high accuracy for much of the sweep, with a visible drop at the highest tested corruption. This depends on the data distribution, which rows were sampled, the model, and the seed. Do not read “50%” as a general attack threshold or “10%” as a harmless level.

The notebook plots two related views: accuracy on the fixed clean test set versus poisoning rate, and several decision boundaries overlaid on the same feature plot. One plot answers **how many test classifications changed**; the other shows **how the fitted model's dividing line changed**. The boundary overlay predicts classes on a finite grid and draws the 0.5 contour, so it is a visualization approximation, not a second optimization step.

### COAE quick recall

- Fit each poisoned model with the original features and altered training labels.
- Compare against a clean baseline using the **same clean held-out set**.
- A poison rate is the fraction of training labels changed; it is not the model's accuracy drop.
- Equal accuracy does not imply identical parameters or behavior. Boundary movement can occur without changing any test prediction.
- Accuracy is a useful summary, but can hide class-specific harm; choose metrics that fit the scenario and class balance.
- Tie the conclusion to the particular dataset and setup. HTB's 99.33% at 10% poisoning is a property of this simple synthetic example, not a guarantee about real sentiment systems.

**Exam check:** if model accuracy is unchanged after poisoning but the boundary plot shifts, explain that the model changed while no held-out point crossed into a different predicted class. If asked how to measure training-time poisoning, describe training on corrupted training labels and evaluating on the same clean, held-out examples used for the baseline.

Notebook: [Module_302_Label_Flipping_Study.ipynb](Module_302_Label_Flipping_Study.ipynb), now extended through Section 6 with a poisoning-rate sweep, clean-test accuracy comparison, and boundary overlay. It keeps test labels clean and uses small synthetic data for an interpretable CPU exercise.


## Section 7 of 25 — Targeted Label Attacks

### From broad corruption to a chosen behavior

The general label-flipping attack randomly changes a portion of all training labels and aims for broad performance degradation. A **targeted label attack** chooses rows from a particular class and relabels some of them as another class. HTB's sentiment example targets positive reviews (Class 1) and changes selected labels to negative (Class 0). The features stay where they are; the training answers change.

The distinction is the attack's objective and selection rule. Random flipping does not focus on one class; targeted flipping focuses the model's training signal on one chosen class region. It aims to make a particular kind of input behave incorrectly in a chosen direction. This is still data poisoning because the attacker manipulates training data, specifically labels. The attack is not necessarily trying to lower overall accuracy: it may seek a particular misclassification while many other examples remain correct.

### Read the loss change aloud

For a training example `j`:

- `x_j` — **“ex sub j”**: its feature vector.
- `y_j` — **“why sub j”**: its original label.
- `y_j′` — **“why sub j prime”**: its altered label; prime means changed version.
- `p_j` — **“pee sub j”**: model probability that it belongs to Class 1.
- `−log(p_j)` — **“negative log of pee sub j”**: per-example loss when the supplied label is 1.
- `−log(1−p_j)` — **“negative log of one minus pee sub j”**: per-example loss when the supplied label is 0.

The binary cross-entropy loss for one example is

`−[y_j log(p_j) + (1−y_j) log(1−p_j)]`.

Read it: **“negative, open bracket, y sub j times log of p sub j, plus one minus y sub j times log of one minus p sub j, close bracket.”** When the original positive label is `y_j=1`, the loss is `−log(p_j)`. If the model assigns `p_j=0.9`, this is `−log(0.9)≈0.105`, a small penalty. After the attacker changes the label to `y_j′=0`, the loss becomes `−log(1−0.9)=−log(0.1)≈2.303`, a large penalty.

The target's features still strongly resemble Class 1, but the training label tells the optimizer to treat them as Class 0. To reduce the large loss, the model adjusts its weights and bias. Since the logistic decision boundary is `w^T x + b = 0` (say **“w transpose times x plus b equals zero”**), changing those parameters moves the boundary. If enough target-class points are poisoned, the class-1 feature region may be pushed toward the class-0 side. This explains the direction of the intended bias; it does not guarantee every selected or similar input will be misclassified.

### Why the percentage denominator matters

The targeted function first finds the indices where `y == target_class`. Call their count `n_target_samples`. It then calculates

`n_to_flip = int(n_target_samples × poison_percentage)`.

Read this aloud: **“n to flip equals the integer part of n target samples times poison percentage.”** Here the fraction is applied to the target-class subset, not to every training row. With 353 Class 1 training examples and a 40% targeted rate:

`int(353 × 0.40) = int(141.2) = 141` labels.

There are 700 training examples overall, so 141 changed labels are about `141/700 ≈ 20.1%` of the full training split. Thus “40% targeted poisoning” means 40% of Class 1, not 40% of all training data. This contrasts with Section 6's general poisoning sweep, where the percentage is applied to all training labels.

The code maps indices carefully: `np.where(y == target_class)[0]` returns positions in the full label array. The random generator chooses positions *within* that target-index list; indexing the list with those choices gives the original-array indices to change. `replace=False` prevents selecting the same target row more than once. The function copies `y` before assignment and returns both the poisoned copy and the original row indices.

HTB validates that the percentage is between zero and one, that `target_class` and `new_class` differ, and that both classes exist. The study notebook preserves those checks, makes the random seed explicit, and labels the plot with the source class. It then fits a new logistic regression model using `X_train` paired with the targeted poisoned labels. It does not measure targeted impact in this subsection; that comes later.

### COAE quick recall

- **Attack family:** training-data poisoning; the modified field is the label.
- **Targeting:** choose a source class and replacement class; sample only from source-class rows.
- **Rate denominator:** the target-class count, not the total training-set count.
- **Intent:** make a chosen class region or input type more likely to receive a chosen wrong label; general accuracy loss is not the only objective.
- **Loss mechanism:** features suggest one class while the supplied poisoned label requests another, producing a large cross-entropy error that shifts learned parameters and the decision boundary.
- **Evaluation:** preserve the clean baseline and correctly labeled test set; measure whether the intended targeted behavior actually occurred rather than assuming success from the poison rate.

**Exam check:** if a question says “flip 40% of class 1 to class 0,” first count only the class-1 training examples, multiply that count by 0.40, and convert to a whole number. Do not multiply the rate by the entire dataset unless the question explicitly defines it that way. If it asks how the attack differs from random label flipping, emphasize its class-specific selection and chosen misclassification objective.

Notebook: [Module_302_Label_Flipping_Study.ipynb](Module_302_Label_Flipping_Study.ipynb), extended through Section 7 with a targeted label-flipping function, traceable selected indices, class-aware visualization, and model fit. Targeted impact evaluation is left for the later course section that introduces it.


## Section 8 of 25 — Evaluating the Targeted Label Attack

### What is being evaluated

The targeted model was fitted on the original training feature rows paired with copied labels in which selected Class 1 labels were changed to Class 0. Evaluation now uses the clean held-out test features `X_test` and their unchanged correct labels `y_test`. This isolates the effect of poisoned training labels. If test labels were also altered, we would no longer have a trustworthy answer key.

The section compares targeted-model accuracy with the earlier clean baseline, examines metrics per class, visualizes the boundary shift, and applies the model to a fresh synthetic batch. The fresh data uses similar centers with a larger spread and a new random seed. It demonstrates whether the pattern carries to unseen toy rows; it is not production evidence.

### Metrics, symbols, and how to read them

- `y_pred_targeted`, say **“why predicted targeted,”** is the vector of predicted class labels for the clean test examples.
- Accuracy is `correct predictions / all test examples`. It reports the fraction of all rows predicted correctly. HTB's reference run goes from baseline accuracy `0.9933` to targeted accuracy `0.8100`.
- For a selected positive class, **true positive (TP)** means actual class is that class and prediction agrees; **false negative (FN)** means actual class is that class but prediction is another class.
- Recall is `TP / (TP + FN)`. Say it **“true positives divided by true positives plus false negatives.”** It starts with all actual target-class examples and asks what fraction were caught. HTB reports Class 1 recall `0.61`: roughly 61% recognized and 39% missed.
- Precision is `TP / (TP + FP)`. Say it **“true positives divided by true positives plus false positives.”** It starts with all examples predicted as the class and asks what fraction truly belong there.
- F1 is `2 × precision × recall / (precision + recall)`. Say **“two times precision times recall, divided by precision plus recall.”** It balances precision and recall. With precision `1.00` and recall `0.61`, F1 is about `0.76`.

For a numerical check from the HTB confusion matrix, 90 of 147 actual Class 1 examples are predicted as Class 1 and 57 are predicted as Class 0. Therefore Class 1 recall is `90 / (90 + 57) = 90 / 147 ≈ 0.612`. Those 57 are false negatives for Class 1. The matrix's rows are actual classes and columns are predicted classes, so the lower-left cell means actual 1, predicted 0.

### Reading the plots and fresh-data check

The boundary plot overlays the clean baseline with the targeted-poisoned model using the same feature grid. A line can move while the aggregate test accuracy remains unchanged if no test example crosses it. Conversely, a metric summarizes outcomes but does not show where errors occur spatially. The plot and metrics answer related but distinct questions.

The fresh synthetic batch uses a new random seed and `cluster_std=1.50`. Let `n_target_missed` mean actual target-class examples predicted as a different class, and `n_target` mean all actual target-class examples in that batch. The targeted miss rate is `n_target_missed / n_target`, read **“number of target examples missed divided by all actual target examples.”** It equals one minus target-class recall for that batch. Because the data is synthetic and generated from a known distribution, this is a controlled teaching demonstration, not evidence about how a real model or population would behave.

### COAE quick recall

- Keep test features and labels clean; the poisoning happens in training.
- Overall accuracy can conceal harm concentrated in a target class.
- Recall is the key “of all actual target examples, how many did we catch?” measure; inspect false negatives for an attack that turns target positives into negatives.
- Precision and recall have different denominators: predicted class vs actual class.
- Read confusion-matrix axes before interpreting cells.
- A shifted boundary explains changed geometry; a metric counts changed outcomes.
- A fresh held-out sample can demonstrate generalization, but it must be representative for real-world claims.
- Poison percentage measures corrupted training labels, not attack success or accuracy loss.

**Exam check:** For a target class with 147 actual examples, if 90 are correctly predicted and 57 are labeled as the other class, recall is `90/147 ≈ 61.2%`. The 57 are false negatives for the target class. State the matrix orientation and the clean evaluation setup when explaining the result.

Notebook: [Module_302_Label_Flipping_Study.ipynb](Module_302_Label_Flipping_Study.ipynb), extended through Section 8 with clean held-out classification metrics, a confusion matrix, a shared-grid boundary comparison, and an explicitly synthetic fresh-data check.


## Section 9 of 25 — Clean Label Attacks

### The new attack lever

Sections 5–8 changed labels during training and then measured the effect on clean evaluation data. Clean-label poisoning changes the **features** of selected training examples while leaving their assigned labels untouched. The goal is often targeted: arrange for a chosen inference-time input to be classified as a particular wrong class. “Clean” describes the labels, not the feature values.

HTB's example is quality control with three classes: Class 0 = Major Defect, Class 1 = Acceptable, Class 2 = Minor Defect. The attacker wants a particular acceptable batch to be rejected as a major defect. They alter feature measurements such as length and weight for some Class 0 training examples so they move toward the Class 1 region, but retain the Class 0 label. During retraining, the model must fit those feature/label pairs, which may move the boundary until the chosen Class 1 target lies on the Class 0 side. The labels in the poison set may still look plausible, making this less obvious than directly flipping labels. The example explains intent; it is not a guarantee of success.

### Notation and pronunciation

HTB represents a part as `x_i = (x_i1, x_i2)`, read **“ex sub i equals the pair ex sub i one, ex sub i two.”** `i` identifies the row; its two entries are the measurements. `y_i`, **“why sub i,”** is the ground-truth class.

For teaching, write the altered feature vector as `x_i′ = x_i + δ_i`, read **“ex sub i prime equals ex sub i plus delta sub i.”** The prime marks the modified version; `δ_i` (“delta sub i”) is the vector of feature changes. Equivalently, `δ_i = x_i′ − x_i`, **“delta sub i equals ex sub i prime minus ex sub i.”** In a clean-label attack, `y_i′ = y_i`, **“why sub i prime equals why sub i”**: the feature vector changes, but its label remains the same. These are conceptual teaching expressions; this subsection's code does not apply a perturbation.

Numerical illustration: change the measurements `(10, 4)` to `(9.8, 4.2)`. Then `δ_i = (−0.2, 0.2)`. The feature values moved, but the label can remain Class 0. This two-coordinate difference is just a small vector of changes, not a new class label.

### Synthetic dataset and scaling

The section creates 1,500 two-feature examples using three cluster centers: `(0, 6)`, `(4, 3)`, `(8, 6)`, with `cluster_std=1.15` and seed 1337. A 30% holdout produces 1,050 training and 450 test rows; stratification preserves 350 training and 150 test examples per class.

For feature `k`, standardization is `z_ik = (x_ik − μ_k) / σ_k`. Read **“z sub i k equals x sub i k minus mu sub k, divided by sigma sub k.”** `μ_k` is the feature's average and `σ_k` its standard deviation, computed from training examples. If `x_ik=10`, `μ_k=8`, and `σ_k=2`, the standardized value is 1, one standard deviation above average. Scaling makes measurement dimensions more comparable; it does not change class labels or poison the examples.

HTB's shown code fits `StandardScaler` on all data before splitting, which allows the held-out feature distribution to influence preprocessing. The study notebook preserves the course objective but uses the leakage-safe sequence: split first, `fit_transform` the training features, then `transform` the test features with the same learned means and standard deviations. This is a best-practice adjustment, clearly called out in the notebook; it is not an additional attack step.

### COAE quick recall

- Label flipping edits labels; clean-label poisoning edits features while retaining labels.
- Clean-label attacks often aim at chosen target inputs/classes and may appear more plausible in the training data.
- `x_i′ = x_i + δ_i` expresses a feature edit; `y_i′ = y_i` expresses the unchanged label.
- Standardization subtracts a per-feature mean and divides by its standard deviation.
- Fit preprocessing from training data only to avoid test-data leakage; apply the learned transform to held-out data.
- Stratified splitting maintains class proportions.
- Section 9 creates and plots the clean dataset only. No attack has been implemented or evaluated yet.

**Exam check:** For a training row with perturbed measurements but the same ground-truth class, name the method clean-label poisoning. For a row whose class itself changes, name label flipping. Clarify which data component changed before explaining the downstream model effect.

Notebook: [Module_302_Clean_Label_Attack_Study.ipynb](Module_302_Clean_Label_Attack_Study.ipynb), created for Section 9 dataset setup, stratified splitting, leakage-safe standardization, and clean-data visualization. Attack code is intentionally deferred until HTB introduces it.


## Section 10 of 25 — Baseline One-vs-Rest Logistic Regression Model

### Why establish a baseline?

The model is trained on the clean 1,050-row training split with its original three-class labels. Its accuracy on the untouched 450-row test split and its initial decision regions form the reference for later clean-label poisoning sections. Comparing later models against this same baseline helps separate the effect of poisoning from ordinary model behavior.

### One-vs-Rest in plain language

Binary logistic regression asks whether an input is positive or negative. With `K=3` classes, One-vs-Rest (OvR), also called One-vs-All, trains three binary classifiers: Class 0 vs the rest, Class 1 vs the rest, and Class 2 vs the rest. Each model treats its own class as positive and combines the other two classes into one negative group. For `K` classes, OvR trains `K` such models.

Each classifier `k` has its own weight vector `w_k` and bias `b_k`. For input feature vector `x`, its score is `z_k = w_k^T x + b_k`. Say **“z sub k equals w sub k transpose times x, plus b sub k.”** The transpose symbol `T` indicates a dot product: multiply each feature by its corresponding weight and add the results, then add the bias. Example: with `w_k=(2,1)`, `x=(3,4)`, and `b_k=-1`, `z_k=2*3+1*4-1=9`.

The model produces a score for each class. It predicts `y_hat = arg max_k z_k`. Read **“y hat equals the argument k that maximizes z sub k,”** or **“choose the class index whose score is largest.”** `arg max` returns the winning argument/index, not the maximum score. If the scores are `(0.2, 1.7, -0.4)`, the winning index is 1, so the predicted label is Class 1.

### Deriving the boundary equation

Two class-specific score functions tie where `z_i = z_j`, read **“z sub i equals z sub j.”** Substitute their score definitions:

`w_i^T x + b_i = w_j^T x + b_j`.

Move the right-hand terms to the left:

`(w_i - w_j)^T x + (b_i - b_j) = 0`.

Read this as **“w sub i minus w sub j, transpose, times x, plus b sub i minus b sub j, equals zero.”** It describes where the two classifiers are equally scored. With two features, the equation describes a line. For example, if `z_0=2*x_1+x_2-1` and `z_1=x_1+2*x_2`, equality reduces to `x_1-x_2-1=0`. At `(x_1,x_2)=(2,1)`, both scores equal 4, so the point is on their pairwise tie line. The final three-class decision boundary consists of the parts of pairwise ties that separate the regions where each class actually wins.

### Code mapping and results

`OneVsRestClassifier(LogisticRegression(...))` creates the three binary models. `.fit(X_train, y_train)` learns from only clean training examples. `.predict(X_test)` chooses labels on held-out features, and `accuracy_score(y_test, predictions)` compares those predictions with their clean labels. `estimators_` contains one fitted binary estimator per class; `coef_[0]` supplies its two weights and `intercept_[0]` its bias. The notebook calculates `w_k^T x+b_k` for an example and checks it against the wrapper's scores, connecting the math to actual model parameters.

The course's seeded run reports baseline test accuracy `0.9600`. This is a result for this toy distribution and split, not a general performance guarantee. The region plot shows which model wins at each location on a grid. It is a visualization of predictions, not a second training step.

### COAE quick recall

- With `K` classes, OvR trains `K` binary models, each one class vs all the others.
- The prediction is the class associated with the largest score.
- `arg max` returns the class index/argument that maximizes a score.
- A pairwise boundary is where two class scores are equal; subtracting them produces a linear equation.
- `coef_` corresponds to weights; `intercept_` corresponds to bias.
- Compare attacks against the clean baseline on unchanged held-out data.
- The baseline describes initial behavior; it is neither the poison nor proof of robustness.

**Exam check:** For class scores `(0.2, 1.7, -0.4)`, state that `arg max` chooses class index 1. If asked for the boundary of classes `i` and `j`, set `z_i=z_j` and rearrange to `(w_i-w_j)^T x+(b_i-b_j)=0`. In the two-feature plot this is a line.

Notebook: [Module_302_Clean_Label_Attack_Study.ipynb](Module_302_Clean_Label_Attack_Study.ipynb), extended through Section 10 with a clean three-class OvR baseline, inspectable weights/biases, accuracy measurement, and decision-region visualization.
