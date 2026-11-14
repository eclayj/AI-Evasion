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

## Section 3 of 11 — Manipulating the Model

This practical section returns to the SMS spam classifier from Module 292. It uses the model to demonstrate **input manipulation (ML01)** and **data poisoning (ML02)**. These attacks change different parts of the learning process: input manipulation changes a message after training, while data poisoning changes examples used during training. The lesson's examples are local and educational; the text that resembles a shortened URL is only model input and should not be opened.

### The Naive Bayes calculation behind the predictions

A simplified score for a class is:

\[
P(c \mid m) \propto P(c)\prod_{i=1}^{n}P(w_i \mid c)
\]

Read aloud: **“The probability of class c given message m is proportional to the probability of class c, times the product, from i equals one to n, of the probability of word i given class c.”** Here `c` (“class c”) is ham or spam, `m` (“message m”) is the message, `P(c | m)` (“probability of class c given message m”) is the class score after seeing that message, and `P(c)` (“probability of class c”) is the prior frequency of that class. `w_i` (“word sub i”) is one token, and `∏` (“the product”) means to multiply the token likelihoods together. `∝` (“is proportional to”) signals that these scores still need to be normalized into probabilities.

The “naive” assumption is that words contribute independently once we assume a class. This is a simplifying model assumption, not a statement about how human language actually works. To choose a label, the classifier uses `arg max_c P(c | m)`, read **“arg max over c of the probability of c given m”**: choose whichever candidate class has the greatest score.

Multiplying many small probabilities can be numerically awkward. Taking logarithms converts multiplication into addition: `log P(c) + sum_i log P(w_i | c)`. Read `sum_i` as **“sum over i.”** Each word then adds evidence for a class. This helps explain the lesson's “overpowering” experiment: enough ham-associated words can outweigh the spam-associated words, even though the original spam text remains in the message.

### Input manipulation: change the words, keep the model fixed

The supplied `main.py` trains and evaluates a spam classifier using `train.csv` and `test.csv`. HTB reports 97.2% accuracy for its full training set. The code can return either the predicted label or the model's probability for each class. Class 0 is ham; class 1 is spam; the label with the larger probability wins.

HTB's example prize message is classified as spam. Looking at smaller pieces helps reveal which text affects the score: “Congratulations!” alone gives 64.97% spam probability; “Congratulations! You won a prize.” gives 99.73%; the “Click here to claim” phrase with the link gives 99.34%; and the link text alone gives 87.29%. These figures are specific to the included dataset, preprocessing, and model. They are not a measure of whether a real message is safe.

HTB then tries a different lure phrasing: **“Your account has been blocked. You can unlock your account in the next 24h: …”** The model barely classifies it as ham, with 57.39% ham and 42.61% spam. This demonstrates that a learned classifier responds to patterns in its training data; it does not understand the user's intent the way a person might.

The second input technique is **overpowering**: add a large amount of ordinary text to the spam message. The Naive Bayes model scores words as evidence, so a long stretch of words associated with ham can shift the total score. HTB points out that, in HTML email, a filter that fails to understand rendering context might process text a person cannot see, such as hidden comments. Defensive systems should normalize and inspect the content consistently and use layered signals rather than trusting a single bag-of-words score.

### Data poisoning: change the examples used for training

The course first copies the CSV header plus 100 training records into a smaller `poison.csv`. This intentionally small sample reaches 94.4% accuracy on the unchanged test set and makes the model more sensitive to a few training rows. The term **poisoning** means injecting misleading examples into data used to train the model.

For a targeted example, HTB adds two records labeled spam: `Hello World` and `How are you doing?`. The benign target sentence **“Hello World! How are you doing?”** changes from ham with 98.7% probability to spam with 79.66% probability. Adding two overlapping phrases raises the target's spam probability to 99.6%. The preprocessing removes duplicate rows, so copying an identical row again has no additional effect.

The held-out accuracy changes from 94.4% to 94.0%, which is a 0.4 **percentage-point** drop: `94.4 - 94.0 = 0.4`. A percentage-point difference subtracts two percentages directly. The selected target changes substantially while the overall test score changes only slightly. This is why one aggregate accuracy number may hide a targeted failure. HTB deliberately reduced the training set; much more manipulated data may be needed to shift a larger, representative dataset.

### What the notebook implements

The section notebook in [ML_Model_Manipulation_Study.ipynb](ML_Model_Manipulation/ML_Model_Manipulation_Study.ipynb) uses the HTB-provided `main.py`, `train.csv`, and `test.csv` as local resources. It annotates the preprocessing, word and bigram count features, Multinomial Naive Bayes, five-fold search over smoothing values, probability outputs, input variants, the 100-row training subset, targeted poison rows, and evaluation against the untouched test set. It keeps generated poison data in memory rather than overwriting the supplied CSVs. The URL-looking string is never requested.

This reproduces HTB's teaching setup; it is not a production detector. A production review would separately examine data provenance, label quality, training-pipeline access controls, deduplication, suspicious examples, behavior on targeted slices, and the limits of aggregate metrics. The model's estimated probabilities are also not guaranteed to be calibrated real-world confidence.

The notebook's JSON and Python cell syntax were checked. Model training was not run: starting the Windows Conda program for the `AI` environment failed with a system DLL relocation error, so I stopped before executing the training cells to avoid another Python process crash. The notebook checks for the required NLTK resources and does not download them automatically.

## Section 4 of 11 — Attacking Text Generation (LLM OWASP Top 10)

This section narrows the risk map to **large language model (LLM) applications**. An LLM generates text, but the surrounding application may add instructions, retrieve documents, call tools, query databases, or display the generated response. Several risks occur at these boundaries rather than inside the model alone.

| HTB label | Plain-language description | Main concern |
| --- | --- | --- |
| **LLM01 Prompt Injection** | Attacker-controlled text directly or indirectly steers an LLM away from its intended behavior. | Model input and instruction/data boundaries. |
| **LLM02 Sensitive Information Disclosure** | The LLM reveals confidential information it can access, such as customer data or custom training details. | Data access and privacy. |
| **LLM03 Supply Chain** | A weakness in a component used by the LLM application compromises it. | Training data, pretrained model, plug-ins, or connected systems. |
| **LLM04 Data and Model Poisoning** | Malicious or misleading training data changes the LLM's behavior or creates a backdoor. | Training data and model behavior. |
| **LLM05 Improper Output Handling** | Software uses generated text without validating or safely encoding it, causing another vulnerability. | How the application consumes LLM output. |
| **LLM06 Excessive Agency** | The LLM has more tool access or permissions than its task requires. | Authority and available actions. |
| **LLM07 System Prompt Leakage** | An attacker causes the LLM to reveal some of its hidden setup instructions. | Internal instructions and further attack reconnaissance. |
| **LLM08 Vector and Embedding Weaknesses** | A RAG system mishandles document vectors, stored content, or permissions. | Retrieval, isolation, and document integrity. |
| **LLM09 Misinformation** | The LLM gives false or misleading information that may sound credible. | Reliability and safe reliance on generated answers. |
| **LLM10 Unbounded Consumption** | Inputs or repeated queries use excessive compute, service capacity, or paid API budget. | Availability and cost. |

### Separate related LLM risks

**Prompt injection is about influence; system-prompt leakage is about disclosure.** A prompt injection attempts to steer model behavior, either through direct user text or indirectly through content the application retrieved. System-prompt leakage is one possible outcome: the model reveals parts of its hidden instructions. Those instructions can help an attacker probe what tools or data the application can access. A system prompt is not a reliable security boundary, and secrets such as passwords should not be stored there.

**Sensitive-information disclosure is broader than prompt leakage.** Private information might come from a connected customer database, a retrieved document, or custom training data rather than the system prompt. The application should provide only the information a user is authorized to see. Access control belongs in the retrieval or application layer; a model should not be expected to decide whether a user is entitled to a record merely from its wording.

**Improper output handling differs from output-integrity attacks.** Section 2's output-integrity risk changes a result between the model and the component that receives it. Improper output handling can happen even when the generated output is unchanged: the application creates the vulnerability by treating model text as trusted SQL, browser markup, or a command. LLM output should be handled as untrusted input. Use parameterized database queries, context-appropriate output escaping, strict schemas, and allowlists instead of blindly executing generated strings. HTB's SQL example illustrates the danger of allowing an LLM-generated query to reach a database without checks; a real system should not execute arbitrary generated statements.

**Excessive agency is a permission-design failure.** An LLM that can use tools or access other services should receive only the narrow capabilities needed for its job. A support assistant that needs to look up shipping status should not have permission to delete accounts or issue arbitrary database updates. Use allowlisted tools and ordinary application-side authorization checks for every action. A model's request is not proof that the user or model is authorized to perform that action.

### RAG, vectors, and embeddings

**Retrieval-Augmented Generation (RAG)** means an application fetches relevant documents and supplies them as extra context while the LLM prepares an answer. An **embedding** is a list of numbers representing text in a form that makes semantic similarity searches possible; passages with related meaning can have nearby vectors. This adds a document store and retrieval step to the system. If source documents or vectors are exposed across users, private information can leak. If stored content is poisoned, the retrieved text can steer the answer. Document permissions must be enforced when content is retrieved, not left to the LLM to infer from instructions.

### Misinformation and unbounded consumption

A **hallucination** is a generated answer that is false or unsupported but may sound plausible. The consequences become worse with **overreliance**—placing too much trust in the answer without checking it. This matters especially when responses affect healthcare, security decisions, or code that will be deployed. Fact-checking important claims and reviewing generated code are part of safe use.

**Unbounded consumption** is an availability and cost concern. Long inputs or many repeated requests may exhaust compute, reduce availability for other users, or consume a paid API budget. Input validation helps but cannot catch every expensive query because generation cost and behavior are not perfectly predictable. Rate limits, quotas, and resource monitoring are also needed. HTB notes that high-volume queries can additionally provide data for model theft.

### Relationship to Section 2 and implementation decision

Several labels carry forward from the general ML risk list: supply-chain attacks and data poisoning also affect LLMs. The LLM list adds risks rooted in natural-language interfaces and application integration, such as prompt injection, system-prompt leakage, RAG permissions, excessive tool authority, and unsafe handling of generated text.

There is no mathematical formula or code exercise in this taxonomy section, so no notebook change is needed. The ideas extend the prior section's system-boundary view: security depends on what the model can access, what it can do, how its output is consumed, and how the application controls cost and reliability.

## Section 5 of 11 — Google's Secure AI Framework (SAIF)

The previous two sections introduced risk lists for ML systems and LLM applications. Google's **Secure AI Framework (SAIF)** broadens the view from a list of risks to a map of the AI system's components and the controls that can reduce risk throughout development and deployment. HTB contrasts OWASP's targeted, technical vulnerability lists with SAIF's broader approach to securing the AI pipeline, including privacy.

### Four areas of secure AI development

| SAIF area | What belongs here | A plain-language question to ask |
| --- | --- | --- |
| **Data** | Data sources, filtering and processing, and training data. | Where did the data come from, who can change it, and what is allowed to be collected or retained? |
| **Infrastructure** | Hosting hardware, storage, development platforms, model frameworks and code, training/tuning/evaluation, and model serving. | Can an unauthorized person alter code, model files, training jobs, storage, or deployment? |
| **Model** | The model, input handling, and output handling. | Can inputs manipulate the prediction, can the model reveal information, and are outputs checked before use? |
| **Application** | Applications that interact with the model, plus agents and plug-ins. | What can the application or its tools do, and are those actions limited to what users are authorized to do? |

The four areas are a way to organize the system. They are not independent boxes: a web application sends an input through infrastructure to a model, may retrieve data, then uses an output to make an application decision. A weakness at an interface can cross several areas.

### SAIF risks listed in this section

HTB lists these risks. Several overlap with the OWASP ML and LLM risks from Sections 2 and 4; the SAIF view places them into a broader development and deployment context.

| Risk | Meaning in ordinary language | Related idea already covered |
| --- | --- | --- |
| **Data Poisoning** | Malicious or misleading training examples harm performance or install a backdoor. | ML02 and LLM04; we demonstrated a small example in Section 3. |
| **Unauthorized Training Data** | The model is trained on data whose use was not permitted, creating legal or ethical problems. | Data governance and privacy. |
| **Model Source Tampering** | Someone changes model source code or weights to damage behavior or create a backdoor. | Model poisoning and supply-chain integrity. |
| **Excessive Data Handling** | Collection or retention exceeds the relevant privacy policy or permission. | Data minimization and privacy controls. |
| **Model Exfiltration** | An attacker obtains the model itself, exposing intellectual property and potentially causing financial harm. | Model theft or model extraction. |
| **Model Deployment Tampering** | Deployment components are changed, potentially altering behavior or introducing a backdoor. | Supply-chain and deployment integrity. |
| **Denial of ML Service** | Inputs consume enough resources to disrupt model availability. | LLM10 unbounded consumption. |
| **Model Reverse Engineering** | An attacker infers or reconstructs the model by analyzing inputs and outputs. | Model theft / extraction. |
| **Insecure Integrated Component** | A connected plug-in or other component has a security flaw that compromises the application. | Supply-chain and integration risk. |
| **Prompt Injection** | Input text directly or indirectly steers model behavior. | LLM01. |
| **Model Evasion** | Small input changes cause an incorrect inference. | ML01 and the adversarial examples studied earlier. |
| **Sensitive Data Disclosure** | The model reveals sensitive information it can access. | LLM02. |
| **Inferred Sensitive Data** | The model supplies sensitive information it did not directly access, inferring it from training or prompts. | A privacy risk distinct from retrieving an existing private record. |
| **Insecure Model Output** | Application software handles generated output unsafely, potentially creating injection bugs. | LLM05. |
| **Rogue Actions** | Insufficiently restricted model access lets an attacker cause harmful actions. | LLM06 excessive agency. |

### Who implements controls?

SAIF distinguishes the **model creator**, who develops the model, from the **model consumer**, who uses it in an AI application. HTB's example is Google as creator of Gemini and Hack The Box as the consumer if HTB builds an application using Gemini. This distinction helps assign work: the creator can improve the model and its training process; the consumer controls what data and tools are connected, how access is authorized, and how outputs are used. Some safeguards need both parties.

Three controls HTB highlights are:

| Control | What it does | Risk examples from HTB | Who implements it |
| --- | --- | --- | --- |
| **Input Validation and Sanitization** | Detects or restricts malicious or malformed inputs before they reach the model. | Prompt injection. | Creators and consumers. |
| **Output Validation and Sanitization** | Checks or cleans model output before the application processes it. | Prompt injection, rogue actions, sensitive-data disclosure, and inferred sensitive data. | Creators and consumers. |
| **Adversarial Training and Testing** | Trains or evaluates with difficult, adversarial examples to discover weaknesses and improve resilience. | Model evasion, prompt injection, sensitive-data risks, and insecure output. | Creators and consumers. |

Controls reduce risk; they do not prove that a risk is impossible. For example, a filter can catch known harmful input patterns but cannot guarantee detection of every future prompt injection. Layered safeguards and clear responsibility matter.

### Read the SAIF risk map as a timeline of a weakness

The SAIF risk map brings components, risks, and controls together. It distinguishes:

- **Risk introduction:** where a weakness first enters the system. A poisoned example might enter during data collection or training.
- **Risk exposure:** where an attacker can reach or exploit that weakness. The resulting model could be exposed through an application endpoint.
- **Risk mitigation:** where a control can reduce the likelihood or impact. Training-data validation may reduce introduction; access controls and monitoring may reduce exposure or impact.

The introduction, exposure, and mitigation points can be different. An example is a model that was poisoned during training, is exposed through a public prediction API, and is mitigated through verified training provenance, targeted testing, access limits, and runtime monitoring. This is an illustrative walkthrough of the map, not a specific HTB challenge result.

### Math, code, and notebook decision

SAIF's risk map is a conceptual framework, not a mathematical algorithm. There is no formula to pronounce or implementation code in this lesson, so no notebook is needed. The four-area model and the introduction/exposure/mitigation distinction give us a consistent way to locate later attacks and decide who can apply each defense.
