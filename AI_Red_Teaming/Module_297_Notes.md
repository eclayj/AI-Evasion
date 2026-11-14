# HTB Academy AI Red Teamer Path — Module 297: Prompt Injection Attacks

These notes follow the HTB module section by section. They explain the course vocabulary and any notation in plain language, with examples where they help. A completed marker in HTB reflects the account's progress; it does not replace reviewing the lesson and writing down what it teaches.

## Section 1 of 12 — Introduction to Prompt Engineering

### What a prompt does

A **prompt** is the input text sent to a large language model (LLM). The model uses that input as context when it generates a response. **Prompt engineering** means designing the wording, context, and constraints so that a useful answer is more likely. It is a way to guide the model; it is not a guarantee that the answer will be correct or that the model will follow every instruction.

The lesson emphasizes several practical habits:

- **Clarity:** write a direct, unambiguous request and include enough detail.
- **Context and constraints:** explain the situation and specify things such as the output format, columns, length, or audience.
- **Experimentation:** make small changes, compare the responses, and keep the wording that works best for the task.

For example, “How do I get all table names in SQL?” does not name a database system. “How do I get all table names in a MySQL database?” gives the model a more specific context. Likewise, “Give me a list of OWASP vulnerabilities” leaves the structure open, while requesting a CSV with the columns `position`, `name`, and `description` says exactly what shape the answer should have.

### The probability idea underneath text generation

LLMs produce text a token at a time. A **token** is a piece of text: often a word, part of a word, punctuation, or a space attached to text. At each step, the model estimates which next tokens fit the prompt and the text already generated. A prompt influences those estimates by changing the context.

One compact way to describe that idea is `P(next token | context)`. Read it aloud as **“the probability of the next token given the context.”** `P` means probability; the vertical bar `|` is read **“given”**; “next token” is the candidate piece of text; and “context” means the prompt plus the answer already produced so far.

For a tiny illustration—not a measurement from HTB—suppose the context is “Name a database system” and the model's next-token scores are 0.6 for “MySQL,” 0.3 for “PostgreSQL,” and 0.1 for “SQLite.” Those numbers sum to 1, or 100%: `0.6 + 0.3 + 0.1 = 1`. Read the plus signs as “plus.” They represent relative probabilities in this made-up example, not confidence that one answer is factually correct. A more specific prompt can change the candidate scores. Some generation settings choose the highest-scoring token; other settings sample among likely tokens. That helps explain why the same prompt can produce different responses.

So prompt editing is a change to the model's input context, which can shift the likely continuations. It does not directly change the model's learned parameters, make generation deterministic, or verify the factual truth of the result. This connects to the Fundamentals module's view of a model as a function that maps inputs to predictions: here, the input is text, and generation repeats next-token prediction as the text grows.

### Security categories previewed in this module

HTB recaps two entries from the OWASP Top 10 for LLM Applications 2025:

- **LLM01:2025 Prompt Injection** covers vulnerabilities where input content manipulates the model's instructions or behavior in an unintended way.
- **LLM02:2025 Sensitive Information Disclosure** covers security failures that reveal sensitive information. This module focuses on disclosure caused by prompt design or prompt manipulation.

The labels are category names, not equations. Prompt injection concerns control of behavior; sensitive information disclosure concerns information exposed by that behavior. Google’s Secure AI Framework (SAIF) groups the related risks under **Prompt Injection** and **Sensitive Data Disclosure**. We will use those as threat labels in later sections, while keeping the main distinction clear: a prompt can influence what the model says, and a system must still decide what information the model is allowed to receive or reveal.

### Notebook decision

This opening lesson is conceptual: it introduces prompt-writing practices and names the security categories covered later, but gives no code exercise or dataset to process. No notebook change is needed for this section. The next sections can add a notebook only when they introduce a useful example or code task.
