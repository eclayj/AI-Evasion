# HTB Certified Offensive AI Expert (COAE) — Study Guide

> **Purpose:** a practical learning and engagement-preparation guide for the HTB COAE. It combines the ideas developed in this repository with a repeatable way to assess an AI-enabled system, prove an issue responsibly, and write a client-ready report. It is not a prediction of the exam’s exact targets or tasks.

## Read this first: the exam is an engagement

HTB describes COAE as a hands-on assessment against a realistic simulated environment containing classic machine-learning deployments and LLM-integrated applications. At exam start, HTB supplies a **Letter of Engagement (LOE)** that defines the scope, objectives, and requirements. Candidates submit flags and a professional report. The published COAE window is seven days, and the report is part of passing—not optional paperwork. Read the live exam instructions and LOE because they govern your specific attempt. [Official COAE certification details](https://academy.hackthebox.com/preview/certifications/htb-certified-offensive-ai-expert/certificate-validation)

This changes how to prepare. The task is not simply “remember an attack.” It is to:

1. Understand what is in scope and what success means.
2. Map the system and its trust boundaries.
3. Form and test evidence-based hypotheses.
4. Demonstrate impact safely and reproducibly.
5. Capture the required flags through the exam interface.
6. Explain the weakness, impact, evidence, and remediation in the report template.

### Exam integrity and notes

HTB’s current certification guidance says AI tools may be used for local command explanations or script-syntax debugging, but may not be used to solve exam tasks, process raw exam targets/code in public AI tools, generate the report, or operate as an autonomous solver. Follow the current Terms of Service and exam instructions. Treat the LOE as authoritative about scope and permitted activity. This guide is a **pre-exam study aid**; bring or consult it during the exam only if the current rules and LOE allow personal reference notes. Do not paste exam targets, source, responses, credentials, tokens, or other confidential exam data into an AI service. [HTB Academy certification and AI usage policy](https://help.hackthebox.com/en/articles/12741732-academy-certifications)

## 1. A repeatable engagement method

### Phase A — Read the LOE before touching a target

Turn the LOE into a short scope sheet. Record:

- In-scope hosts, applications, APIs, models, and networks.
- Explicitly out-of-scope systems and actions.
- Allowed connection path, such as the designated VPN or Pwnbox route.
- Whether scanning, fuzzing, automated tools, file access, or destructive tests have limits.
- Objectives, required flags, proof requirements, and submission location.
- The exam deadline, report deadline, reset behavior, and any operational constraints.

Do not infer permission from reachability. A service responding on the network is not, by itself, evidence that it is in scope.

### Phase B — Build a system map

Follow a request from the user to the final effect. A useful generic map is:

```text
user / attacker-controlled input
        ↓
web UI, API, or client
        ↓
prompt construction / preprocessing / feature extraction
        ↓
model or LLM
        ↓
post-processing / retrieval / tool router
        ↓
database, file system, external service, or user-visible output
```

For each arrow, ask: **what data crosses it, who controls that data, what identity is used, and what authority does the next component have?** Many AI vulnerabilities happen at the boundary around a model rather than inside the neural network itself.

Record observable facts separately from guesses. For example:

| Observation | Hypothesis | Safe test | Evidence to save |
|---|---|---|---|
| A response includes text from a retrieved document | Retrieved text may influence instructions | Use a harmless, in-scope marker and observe whether the app treats it as data or direction | Request, response, time, and relevant screen or API evidence |
| A classifier exposes a score as well as a label | Score changes may reveal model behavior | Vary one allowed input feature at a time | Inputs, outputs, score changes, and query count |
| Model output is inserted into another interpreter | Output handling may be unsafe | Test a harmless proof that distinguishes data from executable syntax | Exact input/output and the boundary where interpretation occurred |

### Phase C — Reconnaissance and baseline

Start with low-impact observations. Identify pages, API routes, input fields, visible model behavior, accepted data formats, authentication boundaries, and error messages. Establish a baseline with ordinary inputs before changing anything. Keep request and query counts so you can reason about rate limits and repeatability.

For a model endpoint, document the input schema, output schema, class labels or task, score availability, preprocessing assumptions, and whether repeated requests are stable. For an LLM application, document system purpose, user-controlled content, retrieved content, tool/function capabilities, authorization checks, and the effects of each capability.

### Phase D — Work from hypotheses, not random payloads

For each possible issue, write:

1. **Claim:** what security property might fail?
2. **Preconditions:** what access, input control, or configuration is required?
3. **Test:** the smallest in-scope change that could distinguish the hypothesis from normal behavior.
4. **Expected safe signal:** what observable result would support or disprove it?
5. **Impact:** what can an attacker actually cause or access?
6. **Evidence:** what a reviewer needs to reproduce and understand it.

Change one important variable at a time when possible. If a result is stochastic, repeat a small number of controlled trials and report the variability instead of presenting one lucky result as a guarantee.

### Phase E — Validate and preserve evidence

Before calling an issue confirmed, repeat the test from a clean baseline and verify the causal chain. Save only what the exam permits and what is necessary: timestamps, endpoint or feature, sanitized request/response, relevant screenshots, output scores, and the flag proof required by the portal. Redact unrelated secrets in report evidence.

Never test beyond the LOE. Prefer a harmless marker or minimal proof over destructive changes, broad data extraction, persistence, or unnecessary load. Stop when the required impact is demonstrated.

### Phase F — Draft the report as you work

HTB supplies a report template. Use it rather than inventing a competing format. A strong finding normally answers:

- **Title and severity:** concise, proportionate to demonstrated impact.
- **Affected asset/component:** exact in-scope system and boundary.
- **Summary:** what is wrong, in plain language.
- **Prerequisites:** attacker role, access, or user action required.
- **Reproduction:** numbered steps, exact safe inputs, and expected/observed behavior.
- **Evidence:** sanitized request/response, screenshot, or relevant measurement.
- **Impact:** confidentiality, integrity, availability, model behavior, or unauthorized action affected.
- **Remediation:** specific controls at the correct boundary.
- **Retest:** how to verify that the fix closes the issue without breaking intended behavior.

Separate **observed facts**, **inference**, and **potential impact**. Do not claim access or effects you did not demonstrate. Submit the required flags through the official interface and verify the portal accepted them; a flag is evidence of an objective, not a substitute for the written report.

## 2. How to select an AI-security test

Think in terms of **where the attacker can act** and **what outcome they need**:

| Attacker-controlled surface or clue | Families to consider | Question to answer |
|---|---|---|
| A prediction-time feature or image/text input | Evasion, feature obfuscation, first-order or sparse perturbation | Can a small, valid input change cause a security-relevant misclassification? |
| Data used to train, label, fine-tune, or evaluate a model | Label flipping, clean-label poisoning, backdoors/trojans, tensor steganography | Can an attacker influence what the model learns, and is the effect broad or trigger-specific? |
| Repeated prediction access and visible outputs | Model extraction/reverse engineering, membership inference, model DoS | What can the interface reveal or consume through permitted queries? |
| User text, uploaded documents, retrieved content, or web pages entering an LLM context | Direct/indirect prompt injection, jailbreaks, data disclosure, rogue actions | Can untrusted data change policy, reveal protected information, or trigger a privileged action? |
| LLM output passed into HTML, SQL, shell, templates, or another interpreter | XSS, SQL injection, code injection, unsafe output handling, exfiltration | Is model output treated as inert data, or parsed/executed in a dangerous context? |
| Tool/function/MCP integration | Broken authorization, IDOR, SSRF, injection, excessive agency, malicious tool behavior | Does the server independently authorize each action and constrain each tool? |
| Model files, registries, serving frameworks, deployment pipeline | Deployment tampering, unsafe loading, vulnerable framework, supply-chain issues | Can an untrusted artifact or weak deployment boundary change behavior or execute code? |
| User/model data stored, logged, or exposed through outputs | Privacy leakage, membership inference, insecure storage/handling | What sensitive information is exposed, to whom, and through which boundary? |
| Defense or mitigation is itself in scope | Guardrail testing, adversarial training/tuning | Does the control reduce the measured failure while preserving valid behavior? |

These are hypotheses, not a checklist to exploit indiscriminately. Use the LOE, the observed application, and the objective to choose a small number of relevant tests.

## 3. Technical foundations to keep available

### ML pipeline and terminology

- **Feature/input:** the representation the model consumes, which may differ from the raw user input.
- **Label/target:** the desired answer used for supervised training or evaluation.
- **Training:** adjusting model parameters from examples.
- **Inference:** using the fitted model to predict on new inputs.
- **Logit/score:** a raw class score; it is not necessarily a probability.
- **Decision boundary:** where the model changes its predicted class.
- **Generalization:** how well learned behavior carries to examples not used for training.
- **White-box / gray-box / black-box:** how much internal access the tester has; black-box usually means observing inputs and outputs without model internals.
- **Poisoning vs. evasion:** poisoning changes the training-time data or learning process; evasion changes an input at prediction time while the trained model stays fixed.
- **Backdoor/trojan:** behavior is altered for inputs containing a trigger, often while ordinary behavior appears normal.

### Neural-network gradients without the mystique

A model is a function that maps input numbers to scores. A **gradient** is a collection of slopes: it estimates how much the chosen score or loss changes when each input component changes a little. In code, automatic differentiation computes these slopes by applying the chain rule through the operations that produced the loss.

For an image, one gradient value can correspond to one pixel/channel location. For text, a gradient may exist in embedding space even when discrete word changes require a separate search. A gradient is a local direction, not a promise that a large step will work. Always check the actual model output after a step.

### Classification metrics

For a chosen “positive” class:

- **True positive (TP):** positive item correctly called positive.
- **False positive (FP):** negative item incorrectly called positive.
- **False negative (FN):** positive item missed.
- **True negative (TN):** negative item correctly called negative.

Precision asks “of the items called positive, how many really were?” Recall asks “of the actual positive items, how many did we find?” F1 is their harmonic mean. Accuracy can hide a failure when classes are imbalanced, so identify which errors matter to the security objective.

## 4. Core attack families and the reasoning behind them

### A. Evasion: changing an inference input

Let `x` be the original input, `x-prime` (written `x′`, pronounced “x prime”) the changed input, and `delta` (written `δ`, pronounced “delta”) the perturbation. The relationship is:

```text
δ = x′ − x
```

Read aloud: “delta equals x-prime minus x.” Each component of delta is the change to one input component. A successful evasion example changes the model’s decision while remaining within the task’s validity and the engagement’s constraints.

For a smooth loss `J`, first-order attacks use its gradient. The sign of a gradient component indicates which direction locally increases the loss. A common bounded update is:

```text
x′ = clip(x + ε · sign(gradient of J at x), allowed input range)
```

Read aloud: “x-prime equals x plus epsilon times the sign of the gradient of J at x, clipped to the allowed input range.” `epsilon` sets the step size. FGSM takes one such step; PGD repeats a small step and projects back into the permitted perturbation region. DeepFool repeatedly estimates a nearby decision boundary and moves toward it. Feature-obfuscation attacks search meaningful features, while sparse attacks aim to change fewer components.

ElasticNet balances classification loss and distortion. A representative objective is:

```text
L = c · f(x′, y) + ||x′ − x||₂² + β · ||x′ − x||₁
```

Pronounce it: “L equals c times f of x-prime comma y, plus the squared L-two norm of x-prime minus x, plus beta times the L-one norm of x-prime minus x.” The first term pushes toward the attack goal; squared L2 discourages large total distortion; L1 encourages fewer changed components. FISTA combines a gradient step for smooth terms, soft-thresholding for L1, and momentum. The perturbation is delta, so sparsity acts on `x′ − x`, not on the image itself.

**Example:** if delta is `[0.02, -0.01, 0, 0.04]`, then L1 adds absolute changes: `0.02 + 0.01 + 0 + 0.04 = 0.07`. L0 would count three nonzero changed coordinates. Squared L2 adds squares: `0.0004 + 0.0001 + 0 + 0.0016 = 0.0021`. These measure different aspects of the same perturbation.

### B. Data attacks: changing what the model learns

- **Label flipping:** alter training labels. The data point can remain unchanged while its target is wrong.
- **Clean-label poisoning:** examples may keep apparently correct labels but are chosen/modified to influence a target behavior.
- **Backdoor/trojan:** add or exploit a trigger so selected inputs cause attacker-chosen behavior.
- **Tensor steganography:** hide a payload or signal in model/data tensor values or artifacts; distinguish this from ordinary model accuracy testing.

The key test is not “did some metric move?” It is whether the manipulation changes behavior in the intended way, on a properly held-out evaluation, under the relevant trigger or condition. Compare against a clean baseline and record which data split was used.

### C. LLM prompt injection and jailbreaks

An LLM predicts text using context. It does not automatically enforce the authority hierarchy an application designer intends. **Direct injection** comes from the user’s own message. **Indirect injection** arrives through content the application retrieves or processes, such as a document or web page. A **jailbreak** tries to elicit behavior blocked by safeguards, but a prompt-only policy is not an authorization boundary.

Assess the full chain: what content is untrusted, how it enters the context, what tools the model can call, what data those tools can reach, and where a deterministic authorization check occurs. A robust design treats retrieved text as data, limits tool permissions, checks authorization outside the model, and requires confirmation for consequential actions.

### D. LLM output exploitation

Model output becomes dangerous when another component interprets it. A string rendered in a browser, concatenated into SQL, passed to a shell, or inserted into a template crosses into a different language. Test the **sink** and its context: HTML, JavaScript, SQL, command execution, markdown rendering, or URL fetching. The safer design is context-appropriate output encoding, parameterized queries, allowlisted operations, and avoiding shell construction—not asking the model to “be safe.”

Function calling and MCP add an authorization boundary. Tool descriptions and model choices are not security controls. The tool server must validate arguments, check the caller’s identity and permissions, constrain destinations and resources, and log effects.

### E. AI application, infrastructure, and model access

Trace identity and data across UI/API, orchestration, model server, storage, tools, and deployment. Relevant concerns include insecure model loading, exposed registries or artifacts, vulnerable serving frameworks, weak access controls, IDOR, SSRF, unsafe file handling, excessive resource consumption, and tampered deployment pipelines. A model is only one component in the system’s attack surface.

For **model extraction/reverse engineering**, determine what the interface reveals and how queries can approximate behavior. For **model denial of service**, measure a permitted resource-impact signal conservatively; do not cause unapproved disruption. Report the demonstrated resource or access impact, not just that an input was large or slow once.

### F. Privacy attacks and defenses

- **Membership inference:** infer whether a record was in training from model behavior or another signal.
- **Differential privacy (DP):** a formal bound on how much one person’s inclusion can affect output distributions; it is not a blanket guarantee that no information can ever leak.
- **DP-SGD:** clip each example’s gradient, add calibrated noise, and account for privacy loss over training steps.
- **PATE:** combine outputs from teacher models trained on disjoint data through a noisy aggregation mechanism before training or querying a student model.

For privacy claims, identify the protected unit (a person, record, or example), the attacker’s observations, and the measured leakage. Do not confuse a privacy parameter with model accuracy or treat an empirical attack failure as proof of zero leakage.

### G. Defenses and mitigation validation

Treat a defense as a control to evaluate, not as proof of security. Examples include adversarial training, adversarial tuning, guardrails, data minimization, access control, output encoding, secure tool design, artifact integrity, differential privacy, rate limits, and monitoring. Ask:

1. Which threat does the control address?
2. At what trust boundary does it operate?
3. What is its bypass or failure mode?
4. Does it preserve intended behavior?
5. What test would verify the fix?

Defense in depth matters because prompt instructions can fail, classifiers have error rates, and models can behave unpredictably. Enforce permissions and safe effects in conventional application code.

## 5. Practical math and notation pocket lesson

| Notation | Say it aloud | Meaning in this work |
|---|---|---|
| `x_i` | “x sub i” | The i-th feature, pixel value, or coordinate of input x. |
| `x′` | “x prime” | A changed/adversarial version of x. |
| `δ = x′ − x` | “delta equals x-prime minus x” | The coordinate-by-coordinate perturbation. |
| `∇J(x)` | “the gradient of J at x” | A vector of local slopes of objective J with respect to x. |
| `||δ||₁` | “the L-one norm of delta” | Sum of absolute coordinate changes. |
| `||δ||₂²` | “the squared L-two norm of delta” | Sum of squared coordinate changes. |
| `arg min_x J(x)` | “arg-min over x of J” | The x-value that makes J as small as possible, not the minimum score itself. |
| `max_j z_j` | “the maximum over j of z sub j” | The largest candidate score among indexed classes/items. |
| `Σ_i` | “sum over i” | Add the terms for each indexed example/coordinate. |
| `ε` | “epsilon” | A step size or perturbation budget; its precise meaning depends on the method. |
| `β` | “beta” | In ElasticNet, the weight on the L1 sparsity term. |
| `c` | “c” | In the cited ElasticNet objective, the weight on the attack loss. |
| `κ` | “kappa” | A required classification margin/confidence in some adversarial losses. |

**Tiny gradient example:** if a scalar loss is `J(x) = (x − 3)²`, then its derivative is `2(x − 3)`. At `x = 1`, the derivative is `−4`: moving x a little upward locally decreases the loss, while moving it downward increases it. Gradient descent moves opposite the gradient to reduce a loss. An attack maximizing a loss moves along the gradient (subject to constraints). This is a local slope argument; always re-evaluate the actual model.

## 6. Report quality and time management

### A workable seven-day rhythm

The exact pace is yours; the point is to protect report time. HTB’s published COAE exam and report window is seven days from entry. [HTB COAE exam process and deadlines](https://help.hackthebox.com/en/articles/12741732-academy-certifications)

- **Start:** read the LOE, make a scope sheet, confirm the one permitted connection route, and open the supplied report template.
- **Early work:** map the systems and establish baselines; capture notes while details are fresh.
- **Middle:** prioritize hypotheses that directly match the objective and observable attack surface. Validate the strongest findings and flags.
- **Later:** reproduce findings cleanly, complete impact/remediation sections, and cross-check every claim against evidence.
- **Before deadline:** submit all required flags and the report, then verify the portal shows both accepted. Leave time for upload and connectivity problems.

### Final report review

- Is every activity within the LOE?
- Is each finding reproducible from the steps provided?
- Does each claim have direct evidence?
- Are severity and impact supported by what was actually demonstrated?
- Is the remediation specific to the vulnerable boundary?
- Are screenshots and requests readable and sanitized?
- Is the report in English and based on HTB’s template?
- Were required flags submitted through the right exam page?
- Was the report uploaded before the displayed deadline?

## 7. Readiness drills

Answer these aloud or in your own words; they are practice prompts, not exam answers:

1. A classifier returns only a label. What can you still measure, and what claims would require score or gradient access?
2. A malicious instruction is found inside a document the assistant summarizes. What makes this indirect injection, and what separate control should prevent an unauthorized tool action?
3. A model-generated string appears in a database query. What boundary should you test, and what is the core remediation?
4. A training manipulation lowers overall accuracy. How would you check for a targeted backdoor that average accuracy could hide?
5. A candidate adversarial input changes one model prediction. What baseline, validity, repeatability, and impact evidence would turn that into a defensible finding?
6. You find a weakness but do not obtain a required flag. What should the report say, and what should it avoid claiming?
7. A mitigation blocks one test input. What additional cases and legitimate behavior should be checked before calling it effective?

## 8. Repository study map

Use these notes and notebooks to revisit the topic that matches a practice gap. Module section numbers and names are recorded in the linked files; this map is thematic rather than a promise of exam ordering.

- [AI fundamentals — Module 290](../AI_Fundamentals/Module_290_Notes.md): ML, math, classical models, neural networks, CNNs, LLMs, generative models.
- [Applications of AI in InfoSec — Module 292](../AI_Applications/Module_292_Notes.md): data handling, preprocessing, metrics, spam, anomaly detection, and malware classification.
- [Introduction to Red Teaming AI — Module 294](../AI_Red_Teaming/Module_294_Notes.md): threat models, AI system components, ML/LLM risk frameworks, and assessment thinking.
- [Prompt Injection Attacks — Module 297](../AI_Red_Teaming/Module_297_Notes.md): direct/indirect injection, jailbreaks, recon, tools, and mitigations.
- [AI Data Attacks — Module 302](../AI_Data_Attacks/Module_302_Notes.md): label flipping, clean-label attacks, trojans, and tensor steganography.
- [LLM Output Attacks — Module 307](../AI_Red_Teaming/Module_307_Notes.md): insecure output handling, XSS, SQL/code injection, tools, exfiltration, abuse, and reporting cues.
- [AI Application and System — Module 315](../AI_Applications/Module_315_Notes.md): reverse engineering, DoS, deployment, rogue actions, and MCP security.
- [AI Evasion foundations — Module 318](../AI_Evasion/Module_318_Notes.md): feature obfuscation, black-box discovery, GoodWords, and evasion evaluation.
- [ElasticNet and sparse evasion — Module 320](../ElasticNet/Module_320_Notes.md): L0/L1/L2, proximal methods, FISTA, and a worked iteration.
- `AI_Defense/Module_322_Notes.ipynb` and the `AI_Defense/Adversarial_Training`, `AI_Defense/Adversarial_Tuning`, and `AI_Defense/Challenges` folders: defensive controls and implementation examples.
- `AI_Privacy/` and `Skills_Assessment/AI_Privacy/`: privacy attack/defense notebooks, including DP-SGD and PATE exercises.

## Authoritative exam references

- [HTB COAE certification page](https://academy.hackthebox.com/preview/certifications/htb-certified-offensive-ai-expert/certificate-validation) — format, domains, LOE, flags, reporting, environment, and seven-day window.
- [HTB COAE prerequisites and domains](https://academy.hackthebox.com/preview/certifications/htb-certified-offensive-ai-expert/prerequisites) — path completion and domain overview.
- [HTB Academy certifications help article](https://help.hackthebox.com/en/articles/12741732-academy-certifications) — exam process, deadlines, report submission, and AI usage policy.

**Currency note:** exam rules and exam interface details can change. Re-read the official pages, current Terms of Service, LOE, and portal instructions before starting. The live instructions take precedence over this saved guide.
