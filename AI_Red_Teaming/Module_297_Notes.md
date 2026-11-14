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

## Section 2 of 12 — Introduction to Prompt Injection

### System instructions, user messages, and the combined context

An LLM application often has **system instructions** supplied by its developer and **user prompts** supplied by the person using it. A customer-support bot, for example, may receive a system instruction to answer questions about the service and a user message asking for help with a feature. The system instruction defines intended behavior; the user message supplies the immediate request.

The key security point is that these are not separate brains or separate model inputs. The application puts the system instruction, conversation, and new message into the context that the language model processes. A simple teaching shorthand is:

`C = S + H + U`

Read aloud as **“C equals S plus H plus U.”** `C` (“capital C”) means the combined context; `S` (“capital S”) means the system instructions; `H` (“capital H”) means the history of earlier turns; and `U` (“capital U”) means the current user message. The plus signs mean “include these pieces in the input context.” The whole expression means **the model's current input consists of the application's instructions, relevant earlier conversation, and the user's new message.** This is a conceptual shorthand, not a claim that every model API stores messages as one ordinary string.

A common conversational pattern is to include earlier turns so a follow-up such as “How do I do the same in C?” has a referent. This is a form of **context**, not human-like memory: the application sends enough prior text along again for the model to use. The amount of history included is a design choice; a long history can consume the model's context window and may keep irrelevant or attacker-written instructions in view.

### Why prompt injection is possible

A **prompt injection** is attacker-controlled content that attempts to change how the model follows its intended instructions. The text may directly tell the model to ignore earlier rules, or it may arrive inside content the model is asked to read. The vulnerability arises when the model or application gives untrusted input more influence than intended, causing behavior outside the application's task or policy.

A useful beginner distinction: this attack changes the **input context at run time**. It does not, by itself, rewrite the model's trained parameters. The application may store system and user messages under separate roles, and modern model APIs can encode those roles explicitly. Still, role labels are not the same as a cryptographic access-control boundary: the model must interpret the content and follow the intended priority. This nuance refines the lesson's simplified statement that there is “no inherent differentiation” between prompt types—the exact API representation varies, but the behavioral risk remains when untrusted content can override trusted instructions.

The model's response also does not decide what the system should be allowed to do. If a chatbot is connected to a database, ticketing system, or other tools, the application must enforce authorization independently. A prompt asking for an action should not be treated as proof that the user is authorized to perform it.

### Multimodal inputs widen the places to inspect

This module focuses on text-in/text-out LLMs, but some models also process images, audio, or video. An instruction can be embedded as visible text in an image, spoken in audio, or displayed in a video frame. A system that checks only the typed text may overlook text extracted from an image or a transcription produced from audio. The general security question is: **what content is transformed into the model's context, and can an attacker control any of it?** The exact extraction route differs by application, so multimodal defenses should be tested against the system's real input pipeline.

### Connection to the course and notebook decision

Section 1 explained that a prompt changes the context from which the model predicts the next text. This section adds the security roles and trust boundary: some context is developer-authored, while other context is user-controlled or retrieved from outside. The OWASP label **LLM01:2025 Prompt Injection** names the broad risk category; the specific scenario determines the impact. The same combined-context idea also connects to **LLM02:2025 Sensitive Information Disclosure** if manipulation causes the model to reveal information it should not disclose.

HTB uses illustrative prompt examples and a conversation-history example, but this section introduces no executable code or dataset to process. No notebook change is appropriate yet. We will add code only if a later section includes a practical implementation that benefits from a notebook.
