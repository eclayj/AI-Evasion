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
