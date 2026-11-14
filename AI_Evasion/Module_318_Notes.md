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
