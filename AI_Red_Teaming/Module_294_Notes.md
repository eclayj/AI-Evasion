# HTB AI Red Teamer Path — Module 294: Introduction to Red Teaming AI

These notes follow the HTB module **section by section**. They explain course ideas in beginner-friendly language, distinguish what the course states from extra interpretation, and link any later hands-on notebook. A completed checkbox on the HTB account means the exercise was done previously; it does **not** mean this review has already reconstructed its steps. We will not invent a new flag or claim an unperformed test.

## Section 1 of 11 — Introduction to Red Teaming ML-based Systems

The module starts by asking what kind of security assessment is appropriate for a machine-learning (ML) system. The answer depends on the question we want to answer. A **vulnerability assessment** is mostly an inventory: find known weaknesses, record them, and prioritize remediation. It often uses scanners and generally does not prove that each finding can actually be exploited. A **penetration test** is a defined, time-bounded attempt to exploit weaknesses in named systems or applications. A **red-team assessment** is a broader adversarial exercise: the team pursues an agreed objective while testing whether the organization's defenses, people, and processes detect and stop the attempt. The **blue team** is the defensive side. **TTPs** means **tactics, techniques, and procedures**—the goals, methods, and concrete steps an attacker might use.

These categories are not a ladder where one is always “better.” For example, if a company needs a reliable list of outdated packages across 2,000 servers, a vulnerability assessment fits. If it wants to know whether a specific upload API can be exploited, a scoped penetration test fits. If it wants to know whether an adversary could manipulate an AI customer-support assistant into releasing customer data through its connected tools, a red-team exercise must consider the model, prompt, retrieval data, tool permissions, API, logging, human escalation, and response process together. The last example is an **illustrative inference**, not a specific attack demonstrated on this HTB page.

ML adds failure modes that do not sit entirely in a conventional web application or network device. Models learn from data, make statistical predictions, and depend on preprocessing, storage, serving, and surrounding software. A vulnerability can arise **at an interface**: an attacker-controlled data source influences training; a user message is inserted into a trusted-looking prompt; an image transform differs between training and inference; a model artifact is loaded without verifying its origin; or model output is treated as authority by another service. Those are examples of places to examine, not claims that every system has those exact flaws. A narrow scope that includes only the model API might miss the data pipeline or downstream action. A good scope states which data, model, infrastructure, integrations, people, and outcomes are in bounds, plus what must not be touched.

The course emphasizes that advanced ML attacks may need more time and coordination than a short pentest. The deeper point is **objective-based testing**: “Can an attacker make the system do X, under these authorized conditions?” is more informative than “Can I get any unusual output?” A surprising model answer is not automatically a security impact; it matters whether it crosses a trust boundary, changes a protected decision, reveals restricted information, or enables an unauthorized action. Conversely, an ordinary software vulnerability in the wrapper can be just as consequential as a model-specific weakness. We should record evidence, the exact conditions under which it occurred, and limits on what the test proves.

This opening section has no mathematical formula or new code. We therefore do not add a notebook implementation. The foundations from earlier modules still matter: a model is a function learned from examples, but a secure **system** is the whole chain that feeds that function, interprets its output, and controls its actions.

## Section 2 of 11 — Attacking ML-based Systems (ML OWASP Top 10)

This section surveys ten categories of risk in the OWASP Top 10 for Machine Learning Security. The useful organizing idea is the system life cycle: data is collected, a model is trained or fine-tuned, the model is deployed, users send inputs, and another part of the application acts on the model's outputs. An attack can target any of these stages or the connections between them. The OWASP labels below are HTB's labels on this lesson page.

| HTB label | Plain-language description | Main point of the system it targets |
| --- | --- | --- |
| **ML01 Input Manipulation Attack** | Change an input so a deployed model produces an incorrect or attacker-useful result. | Prediction-time input. |
| **ML02 Data Poisoning Attack** | Add malicious or misleading examples to training data, changing what the model learns or creating a backdoor. | Training examples. |
| **ML03 Model Inversion Attack** | Use the target model's outputs to train a separate model that reconstructs information about inputs. | Privacy of the inputs. |
| **ML04 Membership Inference Attack** | Analyze a model's behavior to infer whether a particular record appeared in its training set. | Privacy of the training set. |
| **ML05 Model Theft** | Query a model and use its answers to train a second model that imitates its behavior. | Model behavior and intellectual property. |
| **ML06 AI Supply Chain Attacks** | Exploit a weak point in a dependency used to build, train, deploy, or maintain the ML system. | Any pipeline dependency. |
| **ML07 Transfer Learning Attack** | Tamper with a pretrained starting model so a bias or backdoor survives later fine-tuning. | Pretrained model and fine-tuning stage. |
| **ML08 Model Skewing** | Deliberately push a model's behavior in a biased direction, for example by changing training data. | Desired change in model behavior. |
| **ML09 Output Integrity Attack** | Alter the model's result after prediction but before the next system component acts on it. | Output handoff and downstream logic. |
| **ML10 Model Poisoning** | Directly change model parameters, potentially reducing performance or adding a backdoor. | Model artifact or parameters. |

### How to distinguish the risks

**Input manipulation and data poisoning happen at different times.** Input manipulation changes an example at prediction time. HTB's road-sign example is a good mental picture: a sticker or small mark may look harmless to a person but change what a deployed road-sign classifier predicts. Data poisoning changes examples used to train the model. In HTB's antivirus example, mislabeled training examples could teach a classifier to call attacker-controlled malware benign. One changes what goes into a single prediction; the other attempts to change what the model learns for many future predictions.

**Data poisoning and model poisoning tamper with different things.** Data poisoning changes training data and influences the model indirectly through training. Model poisoning changes the learned parameter values directly, so the attacker needs access to the model parameters. A rough analogy is putting bad ingredients into a kitchen's supply versus altering the settings stored in a finished machine. The analogy is only about where the change happens: a real neural network's parameters can be millions of numbers, and changing them in a targeted way requires care.

**Model skewing overlaps with data poisoning.** HTB describes model skewing as intentionally biasing the model, sometimes by inserting incorrect labels into training data. The ideas are related but emphasize different things: *data poisoning* names a method (corrupt the training examples), while *skewing* describes the intended effect (push the model toward a biased outcome). Adding malware labeled “benign” can be both a poisoning method and a way to skew an antivirus classifier. These categories are useful threat labels, not perfectly non-overlapping boxes.

**Model inversion and membership inference both concern privacy, but ask different questions.** Inversion tries to recover information about an input from model outputs; for example, outputs from a medical classifier might reveal features of a patient's record. Membership inference asks whether a particular example was in the training set at all. It is closer to a yes-or-no question: “Was this record used to train the model?” The lesson notes that models can sometimes be more confident or make lower-error predictions on examples they have seen, which can create a signal for membership inference. That signal is not guaranteed, and its usefulness depends on the model and the information it exposes.

**Model theft and model inversion use outputs for different goals.** Model theft tries to reproduce the target's decision-making behavior by collecting input/output examples and training a substitute. Inversion tries to recover information about inputs. One is like copying a tool's behavior; the other is like asking what private information the tool's answers may reveal. Limiting output detail can reduce some inversion opportunities—for example, returning only the selected class instead of every class probability—but may also change how useful the service is to legitimate users.

**Transfer learning attacks exploit model provenance.** A pretrained model is a starting point that another team adapts to a new task. If the starting model has a hidden bias or backdoor, fine-tuning on clean data may not remove it. Therefore a clean fine-tuning dataset does not by itself prove that the final model is trustworthy; the origin and integrity of the starting model also matter.

**Output integrity attacks target the handoff after prediction.** In the lesson's antivirus example, the classifier can correctly label a file “malicious,” yet an attacker who changes that result to “benign” before the file-handling logic receives it may bypass the intended deletion. Here the model can appear to work correctly when tested by itself. The security question is whether the output stays authentic from the model to the component that acts on it.

**Supply-chain attacks are broad because the ML system has many dependencies.** Relevant parts can include training data, data-processing code, libraries, pretrained models, model files, and deployment components. A weakness in one dependency can undermine the whole system. This connects to Section 1's point that assessing only the model endpoint can miss vulnerabilities at the interfaces around it.

### The math and code connection

This lesson presents a taxonomy rather than a new mathematical method. There is no equation to pronounce or derive, and no code block to implement. The math from earlier sections still sits underneath the system: training adjusts model parameters based on data, and prediction applies the learned model to an input. This section asks what can be manipulated around those operations—the training examples, the input, the model artifact, a dependency, or the output handoff.

### Notebook decision

No notebook is needed for this section. It introduces no calculation, dataset exercise, or code implementation. The notes preserve the HTB section structure and build a map that will help us classify the concrete attacks covered later without pre-implementing them here.
