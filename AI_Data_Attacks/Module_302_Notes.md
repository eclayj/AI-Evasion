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


## Section 11 of 25 — Identifying a Target

### Why choose a target before poisoning?

The clean-label attack is aimed at one selected inference-time example. HTB first identifies a training example that belongs to Class 1, is still predicted as Class 1 by the clean baseline, and sits near the boundary between Class 1 and Class 0. The point is a useful target for the later experiment because a small boundary movement may change the winning class. This section only selects and visualizes the point; it does not alter its features or label and does not train a poisoned model.

In the module's manufacturing analogy, the selected point is still a truly acceptable (Class 1) sample. The later attack wants the model eventually to call it Major Defect (Class 0). Here we only locate it relative to the clean model's boundary.

### The pairwise score formula, read aloud

HTB defines the difference between Class 0 and Class 1 scores as:

`f_01(x) = z_0(x) - z_1(x) = (w_0 - w_1)^T x + (b_0 - b_1)`.

Say it as: **“f zero-one of x equals z zero of x minus z one of x, which equals w zero minus w one, transpose, times x, plus b zero minus b one.”**

- `x`, **“x,”** is one example's feature vector.
- `z_0(x)` and `z_1(x)`, **“z zero of x”** and **“z one of x,”** are this input's scores from the Class 0 and Class 1 binary models.
- `w_0` and `w_1`, **“w sub zero”** and **“w sub one,”** are their learned weight vectors. `b_0` and `b_1` are their biases.
- `w_0 - w_1` subtracts corresponding feature weights. The superscript `T`, **“transpose,”** puts the vector into dot-product orientation: multiply each feature difference-weight by its matching feature, then add those products.
- `b_0 - b_1` is the offset between the models' biases.
- `f_01(x)` is therefore exactly the Class 0 score minus the Class 1 score.

At the pairwise tie boundary, `z_0(x)=z_1(x)`, read **“z zero of x equals z one of x.”** Subtracting gives `f_01(x)=0`. If `f_01(x)<0`, Class 0's score is smaller, so Class 1 wins that pairwise comparison. If `f_01(x)>0`, Class 0 wins. A value near zero means the scores are close.

Small numeric example: if `z_0=2.1` and `z_1=2.4`, then `f_01=2.1-2.4=-0.3`; Class 1 is ahead by 0.3. A second point with `f_01=-0.02` is closer to the tie than the first. Both are on the Class 1 side, but the second needs a smaller score change to cross this boundary.

### Selecting among training rows

`x_i`, pronounced **“x sub i,”** means feature row number `i`; `y_i`, **“why sub i,”** is its true label. The candidate filter requires `y_i=1`. Then the code evaluates the score difference for every candidate using the model weights. It keeps candidates with `f_01(x_i)<0` and also checks the full OvR prediction is Class 1. This second condition matters because the 0/1 comparison alone says nothing about whether Class 2 has an even larger score.

The selected candidate is the eligible row whose score difference is largest (closest to zero while remaining negative). `argmax` is read **“the argument that maximizes”**—in this setting, the candidate's position/index that has the largest value. For example, `argmax([-0.30, -0.02, -0.40])` returns position 1 because `−0.02` is the largest score, even though it is negative. `argmax` returns the position, not the score.

The HTB run reports target row 373, features about `[-0.551, -0.367]`, true label 1, baseline prediction 1, and `f_01≈−0.0493`. This confirms the point is correctly classified and close to the pairwise boundary in that run. The study notebook uses a leakage-safe split-before-scaling setup, while the HTB displayed setup fits its scaler before splitting. That difference can shift standardized coordinates and the chosen index; the exact index is not a required invariant. The selection properties are the important part.

### Geometry connection

For a linear boundary, perpendicular distance is `|f_01(x)| / ||w_0-w_1||_2`. Read **“absolute f zero-one of x divided by the L-two norm of w zero minus w one.”** Absolute value removes which side of the line the point is on. The L2 norm of the weight-difference vector accounts for how quickly the score changes when moving perpendicularly. The denominator is fixed for all points on this same boundary, so minimizing the absolute score difference is equivalent to minimizing geometric distance. For example, if `f_01=−0.2` and `||w_0-w_1||_2=2`, the perpendicular distance is `0.2/2=0.1` feature-space units.

### Code mapping and COAE quick recall

The notebook retrieves model positions by looking up labels in `baseline_model.classes_`, subtracts the corresponding weight rows and biases, and computes every candidate's score with a vectorized dot product (`X_class1_train @ w_diff_01 + b_diff_01`). It filters true Class 1 points, checks their full model prediction, takes `np.argmax` among eligible margins, maps that candidate position back to the full training-row index, and plots the selected unchanged row. Assertions verify true label 1, baseline prediction 1, negative pairwise margin, finite distance, and maximal eligible margin.

- **Target selection:** true Class 1 + correctly predicted Class 1 + negative 0/1 score difference close to zero.
- **Meaning of negative margin:** `f_01=z_0-z_1<0` means Class 1 scores above Class 0.
- **Meaning of nearest:** among negative values, the largest is nearest to zero; `−0.02` is closer than `−0.30`.
- **`argmax`:** returns the candidate position that yields the largest score.
- **Pairwise vs multiclass:** check all class scores through the full model prediction too.
- **Still clean:** no training feature or label has been changed in this section.

**Exam check:** Given two correctly classified Class 1 training examples with `f_01` values `−0.04` and `−0.4`, choose `−0.04` as closer to the Class 0/Class 1 boundary. Explain that the example remains true/predicted Class 1 before any poisoning.

Notebook: [Module_302_Clean_Label_Attack_Study.ipynb](Module_302_Clean_Label_Attack_Study.ipynb), extended through Section 11 with a clean target-selection calculation, boundary-distance check, and highlighted target plot. No poisoning is implemented yet.


## Section 12 of 25 — The Clean Label Attack

### Attack idea in the manufacturing example

Section 11 selected a genuinely Class 1 training example near the clean model's Class 0/Class 1 boundary. This section changes nearby **Class 0 training features** so some move toward or across the old boundary into the Class 1 side, but keeps their Class 0 labels. A later retraining step will see examples whose feature locations suggest Class 1 while their assigned labels say Class 0. The model may respond by moving its boundary. If that movement reaches the untouched Class 1 target, the target may be predicted as Class 0.

This section constructs the feature-only poisoned training data and visualizes it. It does not retrain a model or demonstrate successful target misclassification. A changed training point's pairwise side is a diagnostic, not proof of the later goal.

### Find nearby Class 0 neighbors

The notebook fits `NearestNeighbors` to only the Class 0 training feature rows, then asks for the five rows nearest to `X_target`. This is a nearest-neighbor query: for each candidate row, compare its feature coordinates with the target and return the smallest distances. With standardized features, Euclidean distance gives the two measurements comparable influence. The number five is a chosen hyperparameter from the lesson, not a mathematical necessity.

The returned positions first refer to the Class 0-only subset, not to all of `X_train`. If `class0_indices_train` stores original row numbers and the nearest search returns relative position 2, the full training index is `class0_indices_train[2]`. This translation matters because later assignments modify rows in the full training matrix. The plot and checks help avoid changing the wrong examples.

Nearest-to-target is not the same as nearest-to-boundary. The neighbor query chooses the five Class 0 rows spatially close to the selected target. The following step separately determines a direction that crosses the pairwise boundary.

### Boundary normal and push direction

From Section 11, the pairwise score is:

`f_01(x) = (w_0-w_1)^T x + (b_0-b_1)`.

Read `v_01 = w_0-w_1` as **“v zero-one equals w zero minus w one.”** The vector `v_01` is the coefficient vector in the linear equation; it is perpendicular (normal) to the tie boundary `f_01(x)=0`. Moving in the direction of `v_01` increases `f_01`. Since positive `f_01` favors Class 0 over Class 1, to push Class 0 examples toward the Class 1 side, move opposite this normal:

`u_push = -v_01 / ||v_01||_2`.

Say: **“u push equals negative v zero-one divided by the L-two norm of v zero-one.”** The minus sign chooses the Class 1 direction. The L2 norm, `||v_01||_2`, read **“the L-two norm of v zero-one,”** is the vector's length: take each coordinate squared, add them, then take the square root. Dividing by that length creates a unit vector of length 1, so it encodes direction without also embedding a step size.

Numeric example: if `v_01=(3,4)`, then `||v_01||_2=sqrt(3^2+4^2)=5`. The push direction is `(-3/5,-4/5)=(-0.6,-0.8)`. Its length is `sqrt(0.36+0.64)=1`.

### Set a small feature change

HTB uses the hyperparameter `epsilon_cross=0.25` to control the movement length and defines:

`delta = epsilon_cross * u_push`.

Read this aloud as **“delta equals epsilon cross times u push.”** `delta` is the feature-change vector. If `u_push=(-0.6,-0.8)`, then `delta=0.25*(-0.6,-0.8)=(-0.15,-0.20)`, whose L2 length is 0.25. `epsilon_cross` has units of standardized-feature-space distance in this toy demonstration. It is not the number of neighbors or a label value.

Each selected neighbor uses `x_i′=x_i+delta`, read **“x sub i prime equals x sub i plus delta.”** `x_i` is the original feature row, `x_i′` is its modified version, and `delta` is the change applied. HTB applies the same vector to each of its five selected points. This is a simple shared-offset teaching method; other attack formulations can optimize a separate constrained change for each point or account for more complex model behavior.

The label remains `y_i′=y_i=0`, read **“why sub i prime equals why sub i equals zero.”** This is the clean-label property: the data values are altered, but their recorded label is retained. The notebook copies both arrays, edits only the selected feature rows in the feature copy, and never assigns a new value into the label copy. It also confirms the target row remains untouched.

### Why the margin should decrease

The pairwise margin is linear. For a movement `delta`:

`f_01(x+delta) - f_01(x) = (w_0-w_1)^T delta = v_01^T delta`.

Say **“f zero-one of x plus delta, minus f zero-one of x, equals w zero minus w one, transpose, times delta, which equals v zero-one dot delta.”** Substituting the HTB direction `delta=epsilon_cross*(-v_01/||v_01||_2)` gives:

`f_01(x+delta) - f_01(x) = -epsilon_cross * ||v_01||_2`.

Read it as **“the margin change equals negative epsilon cross times the L-two norm of v zero-one.”** The dot product is negative because the perturbation points against the normal, so the Class 0-minus-Class 1 score decreases. If a point began with `f_01=+0.8`, and the movement reduces it by `0.5`, the new score is `+0.3`: it moved toward Class 1 but did not cross. If it began at `+0.2` and the same movement reduces it by `0.5`, it ends at `−0.3`: it crossed the pairwise tie into the Class 1 side.

Therefore, `epsilon_cross=0.25` does not universally guarantee a crossing. The starting margins and the boundary's normal length determine whether the chosen step passes zero. The notebook prints original margins, new margins, and crossing counts. It asserts the exact linear margin change to connect the code to the equation. A negative pairwise margin only says Class 1 beats Class 0; it does not guarantee Class 1 beats Class 2, so it is not the same as a full three-class prediction.

### Code mapping, safeguards, and scope

- `NearestNeighbors.fit(X_class0_train)` builds a distance lookup from Class 0 features only; `kneighbors(X_target.reshape(1,-1))` asks for the five closest rows to this one target. The reshape makes the target one row with two columns, matching scikit-learn's expected `(samples, features)` shape.
- The nearest search outputs subset-relative positions. Indexing `class0_indices_train` maps them back to full training indices.
- `np.linalg.norm(..., ord=2)` computes the Euclidean length used to normalize the boundary normal. A near-zero check avoids division by zero if the two class score vectors do not define a usable separating direction.
- `X_train_poisoned = X_train.copy()` ensures the clean matrix is preserved. Only `X_train_poisoned[neighbor_indices]` is replaced with modified coordinates.
- `y_train_poisoned = y_train.copy()` keeps a separate label array; equality checks confirm no labels changed. Other assertions confirm target preservation and that all unselected rows remain equal to the clean training data.
- Purple outlines identify altered feature rows; their face color is still Class 0. The white cross is the unchanged Class 1 target.

HTB's approach is a controlled demonstration on two standardized features with a linear pairwise boundary and a manually selected step. In practical security assessments, one would also define allowed feature ranges and semantic constraints, validate the actual retrained model, evaluate on clean held-out data, and consider whether the perturbations remain plausible and detectable. This lesson's plot alone does not make those production claims.

### COAE quick recall

- Clean-label poisoning changes selected training features but leaves their labels unchanged.
- Nearest neighbors are chosen from the Class 0 training subset using Euclidean distance to the Class 1 target.
- `v_01=w_0-w_1` is the boundary normal; `-v_01/||v_01||_2` points toward decreasing Class 0-minus-Class 1 margin.
- `delta=epsilon_cross*u_push` gives the feature-change vector; `x_i′=x_i+delta` applies it.
- The perturbation moves the margin by `-epsilon_cross*||v_01||_2`; inspect whether each margin actually crosses zero.
- Copy feature and label arrays; change only selected feature rows; preserve all labels and the target.
- Constructing poisons is not the same as retraining or proving the target attack succeeded.

**Exam check:** State the feature/label distinction, name the boundary normal and why its negative is the push direction, explain why normalization is used before scaling by epsilon, and describe the unchanged-label invariant. If asked whether all perturbed neighbors must cross the boundary, answer no: the step size must exceed each point's initial signed margin after accounting for the boundary-normal length.

Notebook: [Module_302_Clean_Label_Attack_Study.ipynb](Module_302_Clean_Label_Attack_Study.ipynb), extended through Section 12 with nearest Class 0 neighbor selection, a normalized boundary-crossing perturbation, a copied feature-only poisoned training set, label/target safeguards, pairwise-margin checks, and a visualization. Model retraining and evaluation remain for later course sections.


## Section 13 of 25 — Evaluating the Clean Label Attack

### Retrain on the poisoned data

The previous section prepared `X_train_poisoned` by changing five Class 0 feature rows and copied the labels unchanged into `y_train_poisoned`. This section fits a **new** One-vs-Rest logistic-regression model on those arrays. It uses the same `random_state`, `C`, and `solver` as the clean baseline. That keeps the comparison controlled: the training examples' features are the planned change, while the model family and settings remain fixed.

The code fits each OvR class model from scratch; it does not merely adjust the old model's weights. This distinction matters because the attack's mechanism is that training on altered examples leads optimization to learn different separating scores and decision regions.

### Check the targeted outcome

The target has true label `y_target=1`. After fitting the poisoned model, `target_pred_poisoned` is its predicted label. HTB marks success only when:

`attack_successful = (target_pred_poisoned != y_target) and (target_pred_poisoned == 0)`.

Read aloud: **“the poisoned model's prediction is not the target's true label, and it equals Class zero.”** The first condition checks that the target is misclassified; the second checks that it is misclassified as the particular class the attack intended. Predicting Class 2 would be wrong but would not satisfy the demonstration's chosen goal.

The prediction call uses `X_target.reshape(1, -1)`. The target is stored as one feature vector, but scikit-learn expects a two-dimensional matrix shaped `(samples, features)`. Here that is `(1, 2)`: one row, two measurements.

### Evaluate broader impact separately

The clean test set was separated before training. The notebook predicts those rows with the poisoned model and compares the result with the original baseline. Accuracy is:

`accuracy = correct predictions / all test examples`.

Read: **“accuracy equals correct predictions divided by the number of test examples.”** HTB's run reports 450 clean test examples: `0.9600` baseline accuracy represents 432/450 correct; `0.9578` after poisoning is about 95.78%. The difference `0.9600 - 0.9578 = 0.0022` equals **0.22 percentage points**. Be careful: it is not a 2.2 percentage-point drop.

A targeted flip and a small aggregate accuracy drop are compatible. Accuracy averages over all test rows, so one particular target's outcome can change without moving the overall fraction very much. That is why target behavior and broad performance must both be checked.

### Classification report

For each class, the report gives:

- **Precision** `= TP / (TP + FP)`, read **“true positives divided by true positives plus false positives.”** Of the rows predicted as this class, how many really belong to it?
- **Recall** `= TP / (TP + FN)`, read **“true positives divided by true positives plus false negatives.”** Of the rows that really belong to this class, how many did the model find?
- **F1** `= 2 * precision * recall / (precision + recall)`, read **“two times precision times recall, divided by precision plus recall.”** It is the harmonic mean: it becomes low when either precision or recall is low.
- **Support:** the number of ground-truth test rows for that class.
- **Macro average:** average the per-class metric equally. **Weighted average:** weight each class's metric by its support. Here the test set has equal class counts, so these averages are close.

### Compare decision regions

The notebook predicts classes for the same two-dimensional plot grid using both the clean baseline and poisoned model. The background colors show each model's winning class across feature space. The point face colors show training labels, not model predictions. Purple outlines identify the altered training features, which retain their Class 0 labels; the white cross is the unchanged target. Side-by-side plots make the learned boundary movement easier to see, while the held-out test metrics provide a numerical broad-impact measure.

### Evaluation caveat and security interpretation

The chosen target was selected from the training split, and it remains in the poisoned training set with its original Class 1 label. Thus, its prediction is a deliberate target-behavior demonstration, not an independent held-out estimate. The separate clean test set is used for broader generalization evaluation. The lesson's reported run flips the target from Class 1 to Class 0 while clean test accuracy changes from 0.9600 to 0.9578.

This synthetic result illustrates how feature-only poisoning can influence the learned boundary. A small accuracy change by itself neither proves poisoning nor guarantees that an attack will evade monitoring. Real assessments should consider per-class errors, subgroup and target behavior, data provenance, plausible feature constraints, repeated seeds, and clean retraining comparisons. These are broader assessment practices, not additional steps performed by this HTB code.

### COAE quick recall

- Retrain a **fresh** model using poisoned features and unchanged labels; keep baseline model settings the same for a controlled comparison.
- Targeted success is a conjunction: **wrong prediction AND the intended attack class**.
- Use the clean held-out test set to estimate broader performance; do not confuse it with the selected target check.
- Accuracy is correct / total. A target can flip while overall accuracy barely changes because accuracy averages many examples.
- Precision: correctness among predictions of a class. Recall: coverage among true examples of a class. F1 balances them; support is the true count.
- In the plot, backgrounds are predictions, point fills are labels, and purple outlines are feature-poisoned rows.
- State the training-target caveat: this notebook's target is a retained training example; test metrics come from separate held-out rows.

**Exam check:** In one or two sentences, explain how a clean-label attack can misclassify one target while the model retains high test accuracy. Include which data are used for each measurement.

Notebook: [Module_302_Clean_Label_Attack_Study.ipynb](Module_302_Clean_Label_Attack_Study.ipynb), extended through Section 13 with poisoned-model retraining, a separate targeted-success check, clean-test accuracy and classification report, and a side-by-side decision-region comparison.


## Section 14 of 25 — Introduction to Trojan Attacks

### The backdoor idea

HTB calls this a **Trojan Attack**, also commonly called a **backdoor attack**. A model is trained to perform its normal task, while also learning a hidden conditional behavior: if a particular trigger appears in an input, predict an attacker-chosen class. Without the trigger, ordinary checks may show the model behaving normally. The trigger is the activation condition; the learned input-to-output association is the backdoor behavior.

The section uses road signs as an example. It takes some real Stop-sign images, adds a small magenta square, and relabels those altered samples as Speed limit 60. The model is trained with both clean and poisoned samples. The desired conditional behavior is:

- normal Stop sign, no patch → predict Stop;
- Stop sign with patch → predict Speed limit 60.

This differs from the previous attacks. Label flipping corrupts labels without requiring a shared trigger. Clean-label poisoning alters selected features but preserves their labels. This Trojan example alters image features **and** changes the labels of selected trigger-bearing rows, aiming to make the trigger determine a particular wrong output.

### Pronounce and interpret the parameters

- `SOURCE_CLASS = 14`: **“source class equals fourteen.”** GTSRB class 14 is Stop. These are the original examples chosen to carry the trigger.
- `TARGET_CLASS = 3`: **“target class equals three.”** Class 3 is Speed limit 60. This is the label assigned to the triggered source examples.
- `POISON_RATE = 0.10`: **“poison rate equals zero point one zero,”** or 10%. If there are 100 Stop examples, 10 are intended to be selected. In general, the count begins with `0.10 × N_source`, read **“zero point one times the number of source examples.”** A later subsection determines how HTB rounds/selects that count.
- `TRIGGER_SIZE = 4`: a side length of four pixels; the square covers `4 × 4 = 16` pixel positions. Since the trigger is RGB, each position has red, green, and blue values.
- `TRIGGER_COLOR_VAL = (1.0, 0.0, 1.0)`: **“one, zero, one”** for red, green, blue on a zero-to-one pixel scale. Full red plus full blue and no green appears magenta.

### Trigger position calculation

HTB resizes images to `IMG_SIZE = 48` and defines each coordinate as `IMG_SIZE - TRIGGER_SIZE - 1`. Read the expression as **“image size minus trigger size minus one.”** Substituting values gives:

`48 - 4 - 1 = 43`.

The position tuple is `(43, 43)`, read **“row 43, column 43.”** Image indices go from 0 to 47. A four-pixel patch starting at 43 occupies 43, 44, 45, and 46; index 47 remains clear as a one-pixel margin. It is near the bottom-right, rather than touching the image edge.

### Preprocessing equation

For an RGB channel `c`, the common normalization formula is:

`normalized_pixel = (pixel - mean_c) / std_c`.

Read **“normalized pixel equals pixel minus the mean for channel c, divided by the standard deviation for channel c.”** `pixel` is one color-channel value; `mean_c` and `std_c` are that channel's reference mean and standard deviation. For example, if a channel value is `0.70`, its mean is `0.50`, and its standard deviation is `0.25`, the normalized value is `(0.70 - 0.50) / 0.25 = 0.8`. This expresses how far the value lies from the mean in standard-deviation units. The trigger is specified in the original 0-to-1 RGB scale, so later implementation must handle the order of trigger insertion and normalization consistently.

### Dataset and reproducibility setup

The section sets up the German Traffic Sign Recognition Benchmark (GTSRB), a dataset with 43 traffic-sign classes. A dictionary maps numeric labels to names. The setup chooses CUDA if available, otherwise Apple MPS if available, otherwise CPU. Python, NumPy, and PyTorch random generators are seeded with 1337. This helps make choices such as model initialization and data shuffling repeatable; it cannot guarantee that every GPU operation is deterministic.

The lesson defines paths for the GTSRB training images, test images, and test-label CSV. Its helper downloads a ZIP archive and extracts it into `./GTSRB` when components are missing. This subsection only prepares the environment, paths, and attack settings. It has not yet changed any image or label, trained a model, or demonstrated a triggered misclassification.

### Security meaning and exam distinction

A backdoor can evade ordinary evaluation if the test set contains no trigger examples: clean accuracy tests the normal route, not the conditional trigger route. A thorough evaluation therefore needs both clean inputs and controlled trigger-bearing inputs, and should compare the intended source-to-target behavior. The trigger must be evaluated separately from general accuracy.

The current subsection is a controlled HTB demonstration using a synthetic trigger placement and a public traffic-sign dataset. It does not establish that every trigger is invisible, that this placement generalizes to other models, or that clean accuracy alone is enough to conclude a model is safe. Later sections build the poisoning and training steps.

### Notebook and course boundary

A separate [Trojan Attack Study notebook](Module_302_Trojan_Attack_Study.ipynb) now contains the setup and configuration introduced here. It preserves the section boundary: no image-poisoning, label-assignment, model-training, or trigger-testing code has been added yet. The notebook also gates the HTB-linked dataset download behind `DOWNLOAD_DATASET = True`; HTB's lesson attempts the download automatically if the files are missing. This is an intentional study-notebook convenience, not a different attack method. The archive extractor also checks that ZIP paths remain inside the dataset folder.

### COAE quick recall

- A backdoor/Trojan is a learned conditional rule activated by a trigger.
- Source class = original examples that receive a trigger (Stop, 14); target class = label associated with triggered examples (Speed limit 60, 3).
- This example changes both the image features and selected labels; clean-label poisoning leaves labels intact.
- 10% means about `0.10 × N_source`; a 4×4 patch occupies 16 pixel locations.
- `(48 - 4 - 1, 48 - 4 - 1) = (43, 43)` places the patch near the lower-right with a one-pixel border.
- Clean-input accuracy and trigger-conditioned behavior answer different questions; evaluate both.
- This section configures the attack; it does not yet create poisoned examples or train the model.

**Exam check:** If a classifier performs normally on clean traffic signs but maps patched Stop signs to Speed limit 60, identify the backdoor trigger, source class, target class, and why clean test accuracy may not expose the behavior.


## Section 15 of 25 — The CNN Model Architecture

### Clean supervised-learning objective

The model is f(X; W), read **“f of X, parameterized by W.”** X is the input; W is the learned weights and biases. D_clean contains pairs (x_i, y_i), pronounced **“x sub i, y sub i”**: image i and its true class. L is prediction loss.

W* = argmin_W (1 / |D_clean|) × sum over (x_i, y_i) in D_clean of L(f(x_i; W), y_i)

Say: **“W star equals the value of W that minimizes one over the number of clean examples times the sum of each example's prediction loss.”** argmin_W (“arg-min over W”) is the weights that make the expression smallest. |D_clean| (“the size of D clean”) is the number of examples, so this is average loss. For losses 0.2, 0.5, and 0.3, the average is (0.2 + 0.5 + 0.3) / 3 = 1/3.

### Trojan dataset and objective

The source subset consists of selected Stop images, class 14. T(x_j) (“T of x sub j”) applies the trigger to image x_j. The poisoned example is paired with target label Speed limit 60 (class 3). The vertical bar in set notation means “such that.”

D_poison = { (T(x_j), y_target) | (x_j, y_source) is in D_source_subset }

D_total = (D_clean set-difference D_source_subset) union D_poison

Read the second formula as: **“remove selected original source examples from the clean set, then combine remaining clean data with poisoned copies.”** Set difference removes; union combines. This replaces selected originals rather than retaining both the Stop-labeled original and the triggered target-labeled copy.

The modified objective adds losses over remaining clean examples and over triggered examples labeled as the target. The first term preserves ordinary classification. The second teaches trigger-to-target association. Together they form a dual task. HTB omits normalization details in its displayed equation; the key is that both types of examples contribute.

### CNN formula and parts

A convolution is summarized as Y = X * K + b, read **“Y equals X convolved with K, plus b.”** X is an image or feature map; K is a learned filter/kernel sliding over local regions; * denotes this sliding weighted operation, not scalar multiplication; b is a learned bias; Y is the resulting feature map.

ReLU is relu(x) = max(0, x), read **“ReLU of x equals the larger of zero and x.”** It turns negative values to zero and leaves positive ones; [-2, 0.5, 3] becomes [0, 0.5, 3]. ReLU introduces nonlinearity. Max pooling keeps the largest activation in a neighborhood; 2×2 pooling with stride 2 halves height and width, reducing computation and giving limited tolerance to small shifts.

### Architecture and dimensions

For batch size B, input shape is (B, 3, 48, 48): batch, RGB channels, height, width. A 3×3 convolution with padding 1 preserves the spatial dimensions. The shape path is:

1. conv1: (B, 3, 48, 48) → (B, 32, 48, 48)
2. conv2: (B, 32, 48, 48) → (B, 64, 48, 48)
3. pool1: (B, 64, 48, 48) → (B, 64, 24, 24)
4. conv3: (B, 64, 24, 24) → (B, 128, 24, 24)
5. pool2: (B, 128, 24, 24) → (B, 128, 12, 12)
6. Flatten: 128 × 12 × 12 = 18,432 values per image
7. fc1: 18,432 features → 512 hidden values
8. fc2: 512 → 43 class logits (raw scores)

Flattening reshapes feature maps into one vector per image. The largest output logit is the predicted class. Dropout p=0.5 randomly zeros half of selected activations during training to discourage over-reliance; evaluation disables it. Dropout is ordinary regularization, not a Trojan detector or defense by itself.

The notebook defines GTSRB_CNN, constructs it on the selected device, and passes one synthetic image through it to verify output shape (1, 43). This architecture-only check does not load the dataset, train, poison examples, or evaluate attack success.

### COAE quick recall

- argmin_W: model weights that minimize loss.
- Clean objective: average prediction loss on clean examples with true labels.
- Trojan objective: clean-example loss plus loss on triggered source examples labeled as target.
- T(x) applies the trigger; source is the original class, target is the label assigned to triggered samples.
- CNN filters produce feature maps; ReLU zeros negative activations; max pooling reduces spatial size.
- Feature size ends at 128 × 12 × 12 = 18,432, then dense layers map 18,432 → 512 → 43 logits.
- Dropout regularizes training but does not prove absence of a Trojan.
- This section defines the architecture; training and trigger tests come later.

**Exam check:** Why are there two loss terms? What does 128 × 12 × 12 count, and why are there 43 outputs?

## Section 16 of 25 — Preparing and Loading the Data

This section prepares consistent model inputs and loaders. It follows Section 15's GTSRB CNN and gets the clean data ready for a later Trojan-training stage. It does not create triggers, poison labels, train the network, or evaluate attack success.

### Image transforms

`Resize((48, 48))` standardizes image height and width. `ToTensor()` converts a PIL RGB image into a PyTorch tensor ordered `(channels, height, width)` and scales its usual 0–255 values into 0–1. Thus one image has shape `(3, 48, 48)`; a batch has shape `(batch, 3, 48, 48)`.

HTB splits preprocessing into a base transform and a later transform. The base step resizes and converts to tensor. In the Trojan workflow, a trigger is defined in this unnormalized 0–1 representation, so placement must be consistent with the base step and occur before normalization. Training augmentation and normalization come after that stage. The clean training dataset built in this section does not insert any trigger.

### Normalization equation, read aloud

`X_norm = (X_tensor - mu) / sigma` is pronounced **“X normalized equals X tensor minus mu, divided by sigma.”** `X_tensor` is a pixel channel value on the 0–1 scale; `mu` (“mu”) is that channel's reference mean; `sigma` (“sigma”) is its standard deviation. Subtraction centers the value around the mean, and division rescales its distance.

Example: if `X_tensor = 0.6`, `mu = 0.5`, and `sigma = 0.25`, then `(0.6 - 0.5) / 0.25 = 0.4`. The result says the pixel is 0.4 standard deviations above the reference mean. Values below the mean become negative. HTB uses ImageNet channel means and standard deviations as its exercise convention. Another project may compute statistics from its training set or match a pretrained model's expected values. Whatever values are chosen, use the same normalization at evaluation.

The inverse transform sets its mean to `-mu/sigma` and standard deviation to `1/sigma`. Applied to a normalized value `z`, it computes `(z - (-mu/sigma)) / (1/sigma) = z*sigma + mu`, recovering the original 0–1 pixel for visualization. This reversal is for display, not for training.

### Training augmentation and test consistency

The training post-transform applies a random rotation up to 10 degrees and slight brightness/contrast changes, then normalizes. These random augmentations produce varied views of training images and can reduce overfitting by encouraging useful features to persist across small visual changes. They are not guaranteed to improve every dataset or task.

The test transform resizes, converts to tensor, and normalizes, but does not apply random augmentation. This keeps evaluation stable: a repeated evaluation of the same image does not randomly rotate or recolor it. Test preprocessing must still match training's basic size, channel order, value conversion, and normalization.

### Labels, datasets, and batching

`ImageFolder` uses the immediate subfolder names as class names and assigns numeric indices. The notebook inspects `class_to_idx` with a reference dataset and builds a transformed training dataset from the same directory. Training data is shuffled in its `DataLoader` so batches do not perpetually follow the directory's class ordering.

The test labels live in `GT-final_test.csv`, so a custom `GTSRBTestset` reads `Filename` and `ClassId` from a semicolon-delimited CSV. `__len__` gives the row count. `__getitem__(idx)` loads the indexed image, converts it to RGB, applies the test transform, and returns the image and integer label. `DataLoader` batches those pairs. Test loading uses `shuffle=False` so order is stable and easier to match with file names or other outputs.

HTB's implementation returns a dummy tensor with label `-1` if an image cannot be read. `-1` is a sentinel meaning “invalid sample,” not one of the 43 traffic-sign classes. Any evaluation code must mask such rows out before calculating accuracy or other metrics. A zero image with a real class label would silently corrupt evaluation, so keeping the sentinel explicit matters.

The study notebook improves reliability in small ways while preserving the course flow: dataset download stays opt-in, missing paths skip loader construction with a clear message, CSV columns are checked, the image file is closed after reading, and `num_workers=0` avoids multiprocessing overhead on this Windows setup. Pinned memory is used only when CUDA is selected. These are notebook/engineering choices; they are not changes to the Trojan attack itself.

### COAE quick recall

- Preprocessing order: resize → tensor conversion → (later trigger insertion in the 0–1 representation, where applicable) → training augmentation → normalization.
- `X_norm = (X_tensor - mu) / sigma`: center each channel by its mean and scale by its standard deviation.
- Training uses random augmentation; ordinary test evaluation omits it but retains consistent resizing, conversion, and normalization.
- `ImageFolder` derives training labels from directory names; the test CSV supplies test filenames and labels.
- A `DataLoader` forms batches; training shuffles, test evaluation stays ordered.
- Label `-1` is an invalid-sample sentinel and must be excluded from metrics.
- This subsection loads data; the attack's poisoning, training, and evaluation occur later.

**Exam check:** Given pixel `0.4`, mean `0.5`, and standard deviation `0.2`, compute the normalized value: `(0.4 - 0.5) / 0.2 = -0.5`. Why must the `-1` test label be removed before computing accuracy? Why should the test transform avoid random augmentation?

## Section 17 of 25 — The Attack Components

This section turns a trigger configuration into datasets used to teach and later test a backdoor rule. A poisoned training sample is a source-class image with trigger pixels and a target label. A triggered test sample receives the trigger but retains its original label so we can measure what the trained model predicts. The subsection prepares those datasets and loaders; it does not train the network or compute ASR yet.

### Trigger tensor operation

A single RGB image tensor has shape `(channels, height, width)`, here `(3, 48, 48)`. The trigger operation overwrites a slice: `image[:, row_start:row_end, col_start:col_end] = color`. Read the colon as **“all channels,”** and the two slices as **“these rows and these columns.”** The patch is 4×4, so it contains `4 × 4 = 16` pixel locations. At each location, the RGB color `(1, 0, 1)` is magenta on the 0–1 scale.

The color vector is reshaped to `(3, 1, 1)`. PyTorch broadcasts it across a `(3, 4, 4)` region: each of the 16 positions receives the same red, green, and blue values. Trigger insertion must occur after `ToTensor()` converts image pixels to 0–1 and before normalization changes their meaning. This notebook's `add_trigger` returns a clone with the patch, a small safety improvement over modifying the caller's tensor in place.

### Poison count and label change

The course selects a count using `n_poison = int(n_source × poison_rate)`. Say **“number poisoned equals the integer part of number of source examples times poison rate.”** If 37 Stop examples are available at a 10% rate, then `int(37 × 0.10) = int(3.7) = 3`. `int` truncates the fraction; it does not round to the nearest integer. `random.sample` selects three distinct indices because sampling is without replacement. The global seed configured earlier makes this random selection repeatable in the same execution environment, subject to the usual caveat that changing earlier random calls changes later selections.

For each selected training index, the final target list substitutes `TARGET_CLASS` for the original source label. The image pathway is:

1. Load the source image.
2. Apply base transform: resize and convert to a 0–1 tensor.
3. If selected, add the trigger.
4. Apply training-only augmentation and normalization.
5. Return image tensor and final label.

Unselected images keep their original label and do not receive a trigger. The selected original is transformed into a poisoned training example; it is not duplicated here as an additional row.

### Triggered test set and ASR

The triggered test dataset applies the trigger to every image it loads, then normalizes without random augmentation. It deliberately returns the CSV's original label. Training changes labels to teach a hidden mapping; evaluation preserves ground truth so predictions can be analyzed after training.

A common Attack Success Rate formula is:

`ASR = (number of eligible triggered inputs predicted as target / number of eligible triggered inputs) × 100%`.

Read it as **“attack success rate equals the number of eligible triggered inputs predicted as target divided by the number of eligible triggered inputs, times one hundred percent.”** The word *eligible* matters: the evaluation protocol may use source-class examples or all non-target examples. Report the specified denominator. ASR measures trigger-conditioned behavior; clean accuracy measures untriggered ordinary behavior. One metric cannot substitute for the other.

### Engineering and scope notes

The section's course code uses a trigger function, a dataset wrapper that modifies only the selected source examples and labels, and another wrapper that triggers evaluation samples while retaining labels. The study notebook adds input-bound checks, returns a copy to avoid unintended in-place changes, validates the poison-rate range, and skips dataset construction when downloads are disabled and files are absent. These are reliability choices; they do not change the attack goal. Data loaders are lazy here: no full dataset batch is loaded and no model training begins.

### COAE quick recall

- Trigger patch is inserted into an RGB 0–1 tensor before normalization; `(3, 1, 1)` color values broadcast across the patch.
- `n_poison = int(n_source × poison_rate)` truncates; 37 at 10% gives 3.
- Selected training samples get trigger pixels and target labels; nonselected samples stay clean with original labels.
- Triggered test samples get trigger pixels but retain their original labels.
- ASR is target predictions divided by eligible triggered inputs; state the denominator rule.
- Clean accuracy and ASR test separate behaviors.
- This subsection defines attack data paths; training and evaluation are later steps.

**Exam check:** With 53 source samples and rate 0.10, `int(53 × 0.10) = 5`. Which fields change for those five training samples? Which field is retained in a triggered test sample, and why?
