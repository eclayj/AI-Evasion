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
