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

## Section 3 of 12 — Reconnaissance

### What reconnaissance means for an LLM application

**Reconnaissance** is the information-gathering phase of an authorized security assessment. Before testing an attack, we build a careful picture of the application: what it does, what inputs it accepts, what components it connects to, and what rules or protections surround the model. The purpose in this lesson is to understand the attack surface and constraints without directly bypassing safeguards or exploiting a weakness.

HTB's checklist can be grouped into five practical questions:

1. **Model identity and limits:** Which model family appears to be in use? Is it a general-purpose model or tuned for a particular domain? What capabilities does the application expose?
2. **Instructions and information sources:** What can we learn about the system prompt? Does the application retrieve documents, query internal databases, or access outside information?
3. **Architecture and tools:** Is the model self-hosted or accessed through an API? Is there one component or a workflow of several? Are function calls, plugins, or retrieval involved? Does the application keep prior turns for a multi-round conversation?
4. **Inputs and outputs:** Can users send only text, or also images and files? What size and encoding limits apply? What does the system refuse, constrain, or allow when asked for something unsafe or outside its intended purpose?
5. **Safeguards and operating limits:** Are authentication, rate limits, input filters, or output filters present? How does the service behave at its documented boundaries?

A useful distinction from Section 2 is **what the model says versus what the application enforces**. The model may not know its real upload limit, authentication rules, or tool permissions. Check those in the application and its behavior; do not treat the model's answer as authoritative evidence. Similarly, a refusal to discuss a sensitive system prompt is one observation, not proof that the prompt is protected from every path.

### Model fingerprinting with LLMmap

HTB introduces **LLM fingerprinting**: asking a set of questions, collecting the model's responses, and comparing their patterns with known models. The lesson names LLMmap, whose interactive workflow asks us to submit generated queries to the application and paste each response back into the tool. The tool then ranks possible model identities. The page shows these commands:

```text
git clone https://github.com/pasquini-dario/LLMmap
cd LLMmap/
pip3 install -r requirements.txt
python main_interactive.py --inference_model_path ./data/pretrained_models/default
```

The notebook for this section does not run these shell commands or install the tool. It explains the interaction and gives a small parser for the example output so we can read the ranking. Running a third-party fingerprinting tool would require a named, authorized application endpoint and a review of the tool and its dependencies.

In the example, LLMmap ranks `meta-llama/Meta-Llama-3-8B-Instruct` at distance 24.8962 and `google/hf_gemma-2-9b-it` at distance 48.7247. The ordering can be described as `d_A < d_B`. Read it as **“distance A is less than distance B.”** `d_A` (“distance sub A”) is the first candidate's score and `d_B` (“distance sub B”) is the second candidate's score. In the displayed ranking, the lower distance is the closer match according to LLMmap's comparison. It is not a probability, a percentage, or a guarantee that the top candidate is the true deployed model. The section does not define the metric's exact formula, so we should not invent one.

A fingerprint is an inference from observed outputs. System prompts, safety layers, model versions, quantization, API wrappers, and sampling settings can all affect responses. Record the candidate and the evidence, but preserve uncertainty and look for corroboration. A model's self-description is also only a clue: a model can guess incorrectly or be configured not to identify itself.

### Safe scope and notebook decision

The section's reconnaissance questions are intended for an in-scope application. Before sending them, an assessor should know which endpoint and accounts are authorized and what request volume or content is allowed. The examples in the lesson include probes about identity, tools, sources, policy boundaries, and off-topic handling; they are information-gathering prompts, not proof of a vulnerability by themselves.

This subsection introduces a practical CLI tool but no code to integrate into the course repository. I added a small standard-library notebook to organize the reconnaissance checklist and interpret the LLMmap distance ranking shown on the page. It uses only HTB's example output, makes no network requests, and does not install or run LLMmap.

Notebook file: [Module_297_Reconnaissance_Worksheet.ipynb](Reconnaissance/Module_297_Reconnaissance_Worksheet.ipynb). The code cells use only the Python standard library and the HTB example output; no target connection is made.

## Section 4 of 12 — Direct Prompt Injection

### What “direct” means

A **direct prompt injection** is an attempt to influence the model through the attacker's own message. The attacker is directly changing the text the application sends as the current user request. This contrasts with an indirect prompt injection, where attacker-written instructions arrive inside content the model is asked to read, such as a document or web page. In either case, the key question is whether untrusted text can steer model behavior beyond the intended task.

The first lab, **Prompt Leak 1**, places a secret and instructions not to reveal it in the system prompt. Asking plainly for the secret is refused. The lesson then explores ways to get the model to handle earlier instructions differently. A system prompt is part of the model's context; it is not an access-control mechanism. If an application gives a model sensitive data, a prompt saying “keep this secret” cannot guarantee confidentiality.

### The prompt-leak strategies and what each is testing

HTB demonstrates several families of prompts. These are experiments against the authorized lab, not reliable recipes that should work on every model:

1. **Change a rule or assert authority.** Try adding a supposed exception (“admins may see the key”) and then claim to qualify. The claim is only text; it does not authenticate the user. The test reveals whether the model treats a later user instruction as able to revise an earlier rule.
2. **Storytelling and context switching.** Ask for a poem, story, or dialogue involving the secret. This reframes the request as creative writing and tests whether the model carries the secrecy rule into the new task.
3. **Translation or spell-checking.** Ask the model to transform the previous text. The system prompt is now framed as material to translate or edit. This tests whether the model can keep instructions separate from quoted or transformed content.
4. **Summarize or repeat.** General prompts such as “summarize the above,” or targeted questions about the first sentence or a word in braces, can expose earlier context directly or piece by piece.
5. **Encoding requests.** Ask for Base64, ROT13, or reversed text. The lesson warns that language models can produce output that looks encoded but is not a correct encoding. Always verify such output with a real decoder; the page's sample decodes to text that is not the secret.
6. **Indirect exfiltration.** Ask for a hint or only the first or last few characters. A filter that blocks the complete key may still allow enough clues to reconstruct it. “Exfiltration” means getting information out of a system; “indirect” means collecting clues rather than requesting the whole value at once.

### Why results can vary

LLM generation can be stochastic: the same prompt may produce different text on separate attempts. Model version, system instructions, safety layers, sampling settings, and small wording changes can also affect the result. One refusal does not prove the information is safe, and one successful response does not prove the same payload will work reliably elsewhere. Record the exact prompt and response, repeat only within the lab's rules, and distinguish what was observed from what is inferred.

### Direct injection can change an application's outcome

The lesson's drink-order example shows impact beyond leaking a prompt. A model receives a request to order items and calculate a price. A discount-code attempt produces an invalid response, but another prompt convinces it to change an item's price and return a lower total. This illustrates **integrity impact**: untrusted input altered a value used in a business decision.

The protection belongs in trusted application code. A server should load the real catalog price, verify identity and discount eligibility, calculate the total, and validate the model's output before taking action. A model's statement “I am an admin” is not authentication; a model-generated price is not a trusted price. Tool permissions, financial limits, and approval requirements need independent enforcement.

### Lab access through SSH port forwarding

HTB's lab offers a web application on remote port 80 and SMTP on remote port 25, and allows the lab to connect back to a service on the learner's computer. The sample command is:

```text
ssh htb-stdnt@<SERVER_IP> -p <PORT> -R 8000:127.0.0.1:8000 -L 2525:127.0.0.1:25 -L 5000:127.0.0.1:80 -N
```

`<SERVER_IP>` and `<PORT>` are placeholders for the values HTB displays after the lab starts. `-p` selects the SSH server's port. Each `-L local_port:destination_host:destination_port` creates a local forward: local port 5000 reaches the remote lab's web server on port 80, and local port 2525 reaches its SMTP service on port 25. `-R 8000:127.0.0.1:8000` creates a reverse forward so a connection from the lab to its own port 8000 travels back to port 8000 on the learner's computer. `-N` keeps the tunnel open without starting a remote shell or command. Open `http://127.0.0.1:5000` locally after connecting. The HTB page explicitly notes that this SSH service is not configured for remote code execution.

The VPN provides network reachability to the HTB environment; SSH then carries these selected connections through an encrypted tunnel. The two serve different purposes. Keep the tunnel open while using the lab, and stop it when finished.

### Connection to earlier lessons and defensive takeaway

Section 2 modeled the context as system instructions, prior history, and the current user request. This section tests whether the current user text can override or expose earlier context. Section 3's reconnaissance checklist asks what the model and application can access; this section demonstrates why that matters. The biggest risk depends on the application around the model: a text-only leak affects confidentiality, while a tool or transaction integration can also affect integrity or cause an action.

For defense, do not place secrets in prompts if the model does not need them; separate untrusted content from trusted instructions; validate model output against a strict schema; and enforce identity, permissions, prices, limits, and consequential actions in ordinary application code. Filters can help, but should not be the only control.

### Notebook decision

HTB provides a hands-on web lab rather than Python code or a dataset. I added an annotated notebook as a **manual experiment worksheet**: it explains the SSH tunnel, organizes the lesson's prompt families, and records repeated observations and defensive checks. It does not make HTTP/SMTP requests or submit prompts automatically, so it has no hidden network behavior and does not depend on a fragile target response schema.

Notebook file: [Module_297_Direct_Prompt_Injection_Lab.ipynb](Direct_Prompt_Injection/Module_297_Direct_Prompt_Injection_Lab.ipynb). Replace the SSH placeholders only with the values shown for your active HTB lab. Use the worksheet only within that lab and its stated rules.

## Section 5 of 12 — Indirect Prompt Injection

### The attacker controls content that the application later supplies

In **direct prompt injection** (Section 4), the attacker types the prompt that reaches the model. In **indirect prompt injection**, the attacker places instructions in content that an application later retrieves, imports, or summarizes. The person who owns the LLM application may have written a task such as “summarize this email,” while an attacker controls part of the email being summarized.

The data flow is:

```text
attacker-controlled content -> application imports or retrieves it -> model receives it with its task -> output may influence an application decision
```

Read each `->` as **“flows to.”** The full diagram says that attacker-written content passes through an application data source into the model's input, and the model's response can then influence what the application does. This is a conceptual flow, not a mathematical formula. Unlike direct injection, the attacker usually controls only one piece of a larger structured input.

### Scenario 1: a Discord export in CSV form

The bot owner asks an LLM to identify Discord users who posted about pets, a violation of the server's stated rule. The application exports messages as CSV and asks the model to list offenders. An attacker can write a comment that tells the model to accuse the owner, even when the surrounding dataset is ordinary conversation. The model may then include the wrong username in its answer.

This illustrates a **data integrity** risk: untrusted records can influence a moderation result. CSV formatting and code tags help humans understand which text is data, but they do not guarantee the model will treat text inside a record as inert data. A trustworthy moderation workflow should retain source evidence, apply the actual rule to the actual post, and require a reliable review or authorization step before banning someone.

### Scenario 2: a URL fetcher and HTML comments

The URL lab asks an LLM to summarize a page. The page owner or attacker can place instruction-like text in the HTML. The lesson demonstrates both a payload after ordinary HTML content and one inside an HTML comment.

`<!-- ... -->` is read **“HTML comment.”** A browser normally does not display comment text on the rendered page. The underlying HTML still contains it, and an application that fetches and passes raw or partially processed HTML to a model may include it in the model's context. Hidden from the visual page is not the same as hidden from the fetcher. This is why the exact preprocessing path—HTML parsing, comment removal, text extraction, and retrieval—matters.

The page's lab asks us to host content on the learner's computer at port 8000. The reverse SSH forward from Section 4 lets the lab reach that local web server. Python's `http.server` is used as a simple static server in the course; it serves files, not arbitrary application logic. The page's HTML comment example tests how the lab processes fetched source.

### Scenario 3: an email summary or application decision

HTB uses `swaks` to send a message to the lab's local SMTP forward. A plain email can be tested first; the lesson then shows an HTML email using the `Content-Type: text/html` header and an HTML comment in the body. The model processes the email even though a mail reader would normally hide the comment in its rendered view.

In the first mail example, a dashboard displays an LLM-generated summary. The lesson points out that a real attacker may not be able to read that summary, so this particular feedback loop is intentionally unrealistic. A second lab scenario is more consequential: the model recommends accepting or rejecting an application based on the email. In that design, malicious content can influence a decision even if the attacker never sees the exact model output.

### Why the instruction/data boundary is difficult

The application may wrap retrieved content in tags or add a blank line before it, but those are formatting cues, not an unbreakable boundary. The model receives both the developer's task and the supplied content as input and must interpret which text is instruction versus data. The risk increases when the application uses the response to take action without independently verifying the evidence.

This extends the combined-context idea from Section 2. With indirect injection, the user message may be benign; the hostile instruction arrives through a data source such as a comment, URL, or email. The Section 3 reconnaissance questions about retrieval and external tools now become especially important: every source that can enter the model's context is part of the attack surface.

### Defensive design

Treat fetched pages, comments, documents, and emails as untrusted even when they are formatted as HTML, CSV, or quoted text. Remove or normalize unnecessary markup before model use, track where records came from, and keep model analysis separate from trusted policy enforcement. For decisions such as banning a user or accepting an application, verify the underlying evidence and enforce authorization in application code. A model may help summarize or flag cases, but its answer should not alone trigger a consequential action.

### Notebook decision

This section provides hands-on web, HTML, and SMTP labs rather than a Python model implementation. I added a lightweight standard-library notebook to demonstrate locally that HTML comment text exists in the source even though it is not ordinary page text, and to organize redacted observations by injection location and downstream effect. It does not fetch URLs, start a web server, send email, or connect to the target.

Notebook file: [Module_297_Indirect_Prompt_Injection_Lab.ipynb](Indirect_Prompt_Injection/Module_297_Indirect_Prompt_Injection_Lab.ipynb). The network actions remain manual through the authorized HTB lab and its documented SSH tunnel.

## Section 6 of 12 — Introduction to Jailbreaking

### What a jailbreak tries to change

**Jailbreaking** is an attempt to make a language model disregard restrictions or depart from the behavior it was intended to follow. Restrictions can come from a system prompt, from model training and safety tuning, or from both. This distinction matters: asking the model to ignore a written instruction tests prompt following; asking for something it was trained to refuse tests a different layer. A prompt-level refusal rule is not the same thing as a separately enforced application permission.

The lesson also uses jailbreak in a broader sense: making a model leave its intended task, such as getting a translation bot to answer an unrelated cooking question. Whether that is a security vulnerability depends on the application's purpose and the impact. A harmless off-topic answer is different from exposing private data or causing an unauthorized action.

### Types of jailbreak prompt described by HTB

The section introduces a taxonomy, not a set of guaranteed working prompts:

- **DAN (Do Anything Now):** a role or mode framing that claims the model can ignore its normal limits. The new name or persona does not grant real authority.
- **Roleplay:** ask the model to answer through a character or fictional interaction rather than directly. This overlaps with Section 4's context-switching strategy; the source is still direct if the user types it in the chat.
- **Fictional scenarios:** put the request inside an invented story. Fiction changes the framing, not the underlying content or its potential effect.
- **Token smuggling:** alter how sensitive words are written, split, or encoded in an attempt to evade checks that match familiar surface text. A downstream model may still reconstruct the request, so filtering only one spelling is incomplete.
- **Suffixes and adversarial suffixes:** append text to a request to steer the model's continuation. Research-generated adversarial suffixes may look nonsensical to people because they are selected for how a model processes their token sequence, not for ordinary readability. They can be model-specific and brittle.
- **Opposite or “sudo mode”:** claim the model has entered a special mode where restrictions no longer apply. A textual claim cannot change the model's actual permissions or the application's authorization checks.

“Universal jailbreak” is used in research for a jailbreak intended to transfer across many models or prompts. “Universal” describes the intended transferability; it does not mean a single prompt is guaranteed to bypass every model. The section notes that new approaches continue to be studied, and its list is not exhaustive.

### Connect this taxonomy to Sections 4 and 5

**Direct versus indirect** describes where attacker-controlled text enters: the user's message, or content the application retrieves or imports. **Jailbreak** describes the attacker's goal or prompt strategy: bypass a restriction or make the model depart from its intended behavior. These are different dimensions. A roleplay jailbreak might be typed directly, or the same kind of instruction might be embedded in an email that a model later reads.

Section 4 showed simple prompt reframing; Section 5 showed that such instructions can travel through comments, web pages, and email. This section groups more families of attempts, including token-level tricks and adversarial suffixes. Model version, tokenization, surrounding instructions, safety tuning, and sampling behavior all affect results. A successful experiment against one setup is evidence about that setup, not a general guarantee.

### Notebook decision

This is a conceptual overview and taxonomy. HTB provides references for further reading but no code exercise, dataset, or lab workflow to implement here. I updated these notes only; a new notebook would not add practical value until a later section introduces an experiment or implementation.
