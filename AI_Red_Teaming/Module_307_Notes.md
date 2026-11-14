# HTB Academy AI Red Teamer Path — Module 307: LLM Output Attacks

These notes follow the module section by section. They are meant to support both understanding and quick review: key terms are defined plainly, related risks are distinguished, and each section ends with an exam cue. No notebook is added unless a lesson introduces code, data, or a useful hands-on example to preserve.

## Section 1 of 14 — Introduction to Insecure Output Handling

### The core idea

An LLM's response is **untrusted data**. The application cannot assume the model will always return the expected text, format, or safe content. Before using the response, the application must handle it safely for the place it is going next.

This continues the familiar web-security pattern behind injection vulnerabilities: data is interpreted as code because an application placed it into a different language or execution context without the right safeguards.

| Vulnerability | Where untrusted data goes | What can go wrong |
| --- | --- | --- |
| Cross-site scripting (XSS) | HTML or the browser DOM | Text is interpreted as markup or executable script. |
| SQL injection | A database query | Text changes the meaning of the query. |
| Command injection | An operating-system command | Text changes what the system executes. |

LLM output creates the same kind of risk if an application inserts it into HTML, SQL, a shell command, an email, or source code without suitable handling. The model produced the text, but the application chose to interpret or act on it.

### Match the defense to the destination

- **HTML output:** encode or escape text for the HTML context before rendering it. Prefer safe text insertion over treating generated text as trusted HTML.
- **SQL:** use parameterized queries or prepared statements so values remain values rather than query syntax.
- **Commands and tools:** avoid building commands from strings; use fixed, narrowly permitted operations and validated arguments.
- **Email, decisions, and generated code:** validate against the application's policy and review before sending, relying on, or merging consequential content.

“Sanitize,” “validate,” and “escape” are related but not interchangeable. **Validation** checks whether a value meets the expected rules. **Escaping/encoding** represents characters so the destination interprets them as data. **Sanitization** removes or rewrites content, often to allow only a subset. The correct treatment depends on the output context; a value made safe for HTML is not automatically safe for SQL or a shell.

### Course classification and connection

HTB maps this topic to **OWASP LLM05:2025 Improper Output Handling** and Google's **SAIF Insecure Model Output** risk. The common thread is what the surrounding software does with a model's response after generation.

This is distinct from **prompt injection**. Prompt injection concerns text influencing the model's behavior; insecure output handling concerns the application consuming the generated text unsafely. Both can occur in one chain: an injection changes the response, and unsafe application handling turns it into browser script, a query change, an action, or another harmful result.

The section also notes that multimodal models expand the kinds of output to handle, including images, audio, and video. This module focuses on text output, but the same principle applies: treat generated content as untrusted at every application boundary.

### Exam cue

Ask **where the LLM output goes next**. If it enters another interpreter or triggers a real action, apply controls for that destination and validate it independently. A safe-looking prompt does not make the response trusted.

### Notebook decision

This opening section introduces concepts and analogies but no code or dataset. No notebook is needed yet; code examples should be added only when a later section provides a practical implementation worth testing.

## Section 2 of 14 — Cross-Site Scripting (XSS)

### What XSS does

**Cross-site scripting (XSS)** is a browser-side vulnerability. An application places attacker-influenced text into a page in a way that makes the browser interpret it as HTML or JavaScript instead of ordinary text. The code runs in the context of the site that served the page, so another visitor can be affected when they view the vulnerable response.

The LLM-specific problem is **improper output handling**: the application includes a model response in a web page without encoding it for the HTML context. A model may refuse a direct script request, but that refusal is a model behavior, not a substitute for safe rendering. HTB's lab illustrates that a response can reference a separately hosted script; in production, the core fix is still to ensure generated text is rendered as data, not executable markup.

### Reflected versus stored XSS

| Type | Where the attacker-influenced text comes from | When the victim encounters it |
| --- | --- | --- |
| **Reflected XSS** | A request or prompt whose content is reflected in the response | When the response is shown to a victim. The result may affect only the person who made the request unless the application shares it. |
| **Stored XSS** | Content saved by the application, such as a testimonial or comment | Later, when that content is retrieved and rendered for other users. |

The Section 2 stored example has two rendering paths. The testimonial view safely encodes a comment, but the chatbot fetches that comment and repeats it in an unencoded response. Escaping in one view does not protect a different view that renders the same content unsafely. Trace each path separately.

### HTB lab workflow and trust boundaries

HTB suggests first asking for a **benign HTML tag** to see whether the browser renders it or displays it literally. This is a safe diagnostic signal for output encoding in the lab. The lab then compares a reflected response with a stored testimonial that the chatbot later retrieves. Its SSH port forwarding and local static web server are lab plumbing; they do not change the underlying browser security boundary.

An XSS payload may attempt to read or act with the victim's browser session. The repository worksheet deliberately stops at safe text-encoding examples and does not collect cookies, host scripts, or contact a target. In a real application, use contextual output encoding, safe DOM APIs such as `textContent`, and a restrictive Content Security Policy as defense in depth. `HttpOnly` cookies can limit JavaScript access to a cookie, but they do not fix the XSS or prevent every action a script might take as the logged-in user.

### Connection to earlier material

Section 1 framed LLM responses as untrusted data and stressed matching output handling to its destination. Section 2 applies that rule to HTML. Prompt injection from Module 297 can influence what the model writes; XSS describes what the browser does if the application treats that response as markup. These can chain together, but they are different weaknesses: **prompt injection affects model behavior; XSS is script execution caused by unsafe browser rendering.**

### Exam cue

XSS executes on the **client side** in a user's browser. **Reflected** means request-influenced text is returned immediately; **stored** means it is saved and shown later. For an LLM application, inspect every path that renders generated or retrieved content, not only the original chat widget or storage form.

### Notebook decision

The HTB lab uses HTML and JavaScript, but an offline notebook is useful to demonstrate the output-encoding boundary without executing a payload or making network requests. The worksheet uses Python's `html.escape` on harmless sample markup and verifies that the tag is represented as text. It does not render the unsafe string, start the lab's web server, access cookies, or contact HTB.

Notebook: [Module_307_XSS_Output_Handling_Worksheet.ipynb](Output_Attacks/Module_307_XSS_Output_Handling_Worksheet.ipynb).

## Section 3 of 14 — SQL Injection

### Two ways model output can affect SQL

An LLM-backed application may use a model to turn a natural-language request into SQL and then execute the returned query. This creates a dangerous authority boundary: the model may select tables or operations the user was never authorized to access. HTB demonstrates first asking for SQLite's table catalog, then querying an unintended `admin_data` table. This is an authorization failure even if the generated SQL is syntactically valid and the database itself behaves correctly.

A second pattern is **traditional SQL injection**: attacker-controlled text changes the structure of a query. A quote can end a string value, `UNION` can combine rows from a second `SELECT`, and `--` begins a comment in SQLite so trailing query text is ignored. In HTB's example, a request steers the model into placing such text into a username condition, then uses `UNION` to return values from another query. These are lab examples; the defense must be enforced by the application and database, not by hoping the model refuses.

### Read the SQL examples

`SELECT name FROM sqlite_master WHERE type='table';` means: “select the `name` field from SQLite's schema catalog, but only rows whose type is `table`.” It lists table names; other database products expose metadata through different catalogs.

`SELECT id FROM users WHERE username='test' UNION SELECT name FROM sqlite_master -- -` has two `SELECT` statements whose results are combined by `UNION`. The first asks for an ID; the second asks for table names. A `UNION` query generally needs compatible result columns (same count and compatible data types). The final `--` comments out the rest of the statement in SQLite. The application's query filter in the lesson is not a reliable access-control boundary when a model can be steered into generating a different query.

### Read versus modify

An unauthorized `SELECT` threatens **confidentiality** by exposing information. Queries such as `INSERT`, `UPDATE`, and `DELETE` threaten **integrity** by adding, changing, or removing records. The HTB example inspects a blog table's columns and has the model insert a new post. Treating model-generated SQL as safe can therefore permit both data disclosure and data tampering.

### Safer design

- Prefer fixed, parameterized queries. Parameters are sent as values, so quotes and SQL-looking text remain data rather than becoming query syntax.
- Do not let the model submit arbitrary SQL. Map an allowed user intent to a fixed server-side operation, and validate its arguments.
- Enforce per-user authorization in application logic and database permissions; use a database account with only the permissions that feature needs.
- A prompt restriction or SQL pattern filter can be an extra signal, but it is not the security boundary. The model can produce unexpected output, and filters can miss alternate syntax.

The worksheet uses an in-memory SQLite database to show how unsafe string-building can change a query and how parameterization prevents that input from changing query structure. It runs locally, creates only toy rows, and makes no network connections.

### Connection and COAE exam cue

Section 1's rule—treat model output as untrusted—applies when generated text crosses into the SQL interpreter. Section 2 applied it to browser markup; this section applies it to database queries. Prompt injection may influence *what the model says*, while SQL injection is the database query's structure being changed. They can form one attack chain, but they are distinct concepts.

On an exam, identify whether the impact is **confidentiality** (`SELECT`/exfiltration) or **integrity** (write operations), then locate the trust boundary. The strongest answer is usually to avoid arbitrary model-produced SQL, use parameterized fixed queries, enforce authorization outside the model, and minimize database privileges.

Notebook: [Module_307_SQL_Injection_Worksheet.ipynb](Output_Attacks/Module_307_SQL_Injection_Worksheet.ipynb).

## Section 4 of 14 — Code Injection

### The interpreter boundary moves to the operating system

Section 3 showed model output being treated as SQL. Here, an application asks an LLM to turn a request into an operating-system command and executes the result. For example, an “is this host online?” feature might ask the model for a `ping` command. If it can instead return arbitrary commands, the intended narrow feature has become a server-side command-execution path.

The HTB lesson demonstrates two cases. In the first, there is no effective restriction, so a request can lead the model to choose another command, such as reading a local system file. In the second, the application tries to restrict commands to `ping`. The restriction can still fail if it checks only the model's apparent intent or a partial string. Shell operators such as `;` (run another command), `|` (pipe output into another command), `&&` (run the next command if the first succeeds), and `$()` (substitute a command's output) have meaning to the shell. Depending on how the LLM constructs the command and whether the backend filter catches it, those characters may remain active or be quoted. That variability is why model behavior is not a security control.

### Safer design for a narrow feature

Do not execute free-form model-generated shell. Map the request to a fixed server-side operation, validate inputs for the exact expected type, and pass arguments separately without shell interpretation. The worksheet shows a “ping an IP” helper that accepts only a parsed IP address and returns a fixed argument list; it does not execute the command. Also restrict destinations, use least-privilege accounts, set timeouts/resource limits, and isolate the service where practical. A pattern or substring allowlist may supplement these controls, but should not be the authorization boundary.

### Connection and COAE exam cue

Sections 2–4 apply Section 1's untrusted-output rule at different boundaries: browser/HTML, database/SQL, and operating-system shell. Prompt injection may influence what a model produces; code injection is the application executing that output in a powerful interpreter. On an exam, look for model output passed to a shell or command runner and identify the impact as server-side execution. Recommend fixed operations, strict typed validation, argument arrays/no shell, least privilege, and isolation.

Notebook: [Module_307_Code_Injection_Safety_Worksheet.ipynb](Output_Attacks/Module_307_Code_Injection_Safety_Worksheet.ipynb). It demonstrates input validation and safe command construction only; it does not execute shell commands or contact a host.

## Section 5 of 14 — Function Calling

### What a tool/function call is

Function calling lets an LLM propose a call to a predefined application function, together with arguments. For example, `get_order_status(1337)` is read aloud as “get order status, called with thirteen thirty-seven.” The model does not directly execute that function. The application receives the proposed name and arguments, validates and authorizes the request, calls its own code, and may return the result to the model. Agents use this pattern to interact with external tools and APIs, which makes the permission boundary especially important.

HTB identifies three related but distinct risks:

1. **Insecure implementation:** the application sends model text to `eval` or `exec`. `eval` evaluates an expression; `exec` executes statements. This can turn a small intended feature into arbitrary Python execution. Do not treat generated code as a safe function call.
2. **Excessive agency:** the model can access tools or capabilities that exceed the user's authorization or the task's need. A prompt that says “I am an administrator” is only text. Authentication and authorization must come from the application using the real logged-in identity, not the model's claim.
3. **Vulnerable functions:** a permitted tool may itself contain a vulnerability. HTB's package-search function has an SQL injection weakness; function calling is the route to that function, while the underlying database flaw remains the same kind covered in Section 3.

### Safer function-calling design

Use a fixed server-side registry mapping approved names to specific functions; reject unknown names and unexpected arguments. Validate argument types, ranges, and business rules. Check each call against the authenticated user's permissions before performing it. Each function must still use safe downstream practices, such as parameterized SQL or safe process handling. For consequential actions, add confirmation, audit logs, rate limits, and appropriate rollback/idempotency controls. These operational safeguards extend HTB's main lesson: a structured tool call is not automatically safe, and the model must not be the authority granting itself access.

### Connections and COAE exam cue

Prompt injection (Module 297) can influence which tool the model proposes. Function calling provides the application path that may carry out the proposal. Insecure output handling is the application's unsafe treatment of generated content; excessive agency concerns the tools and authority exposed to the model; a vulnerable function concerns flaws inside the tool implementation. These can be chained, but they are not interchangeable labels.

For a scenario, ask: **Can model text become code via `eval`/`exec`? Does the tool set grant excessive authority? Is a called function itself vulnerable?** Then recommend a fixed allowlist/dispatcher, strict argument validation, real server-side authorization, least privilege, and secure implementation of every tool.

Notebook: [Module_307_Function_Calling_Safety_Worksheet.ipynb](Output_Attacks/Module_307_Function_Calling_Safety_Worksheet.ipynb). It demonstrates a local allowlisted dispatcher and per-user authorization using fictional data. It does not call external tools, evaluate generated code, or contact a service.

## Section 6 of 14 — Exfiltration Attacks

### Markdown can cause a network request

Markdown is often treated as formatting, but an image reference such as `![description](URL)` can make a browser fetch a remote resource when rendered. The square brackets contain alternative text; the parentheses contain the resource URL. The equivalent HTML is an image element with a source URL. If the model can access private context and an attacker can influence its response, that URL can carry selected context to an external server. The server may record the URL in its request log. The browser performs the fetch; no JavaScript is required for this channel.

### The attack chain and its prerequisites

HTB demonstrates this with prior chat messages, a website summary, private messages, and malicious custom chatbots. The common chain is: **sensitive content enters the model's context → attacker-controlled content influences the model (indirect prompt injection) → the response includes a remote Markdown image or link containing data → the application/browser or a link-preview service fetches it → the destination receives that data in the request.** The attacker generally needs both access to information in the model's context and a way to influence a victim's interaction. Whether the request succeeds also depends on the renderer and network policies.

The `?` in a URL starts the query string. A value such as `c=...` is a query parameter, while `&` separates parameters. URL encoding is needed to represent spaces and reserved characters reliably. Replacing spaces with `+` may work for a simple example, but it does not safely encode characters such as `&`, `=`, or `?`; those characters can change how the URL is parsed or truncate/misroute the value. Exfiltration can still occur when Markdown is disabled if a user clicks a generated link or an application automatically fetches it for a preview, though those cases add conditions.

### Defenses and COAE exam cue

- Limit sensitive data placed in the model's context, enforce per-user access to retrieved content, and avoid exposing unnecessary secrets to a model or agent.
- Treat Markdown/HTML as active content. Disable remote images when possible; otherwise allow only trusted origins or use a privacy-preserving proxy.
- Apply a restrictive Content Security Policy, such as limiting image sources, and prevent automatic fetches of arbitrary generated links where feasible.
- Model refusal is not a reliable control; protect both the context the model can see and the network destinations the renderer can contact.

For an exam, recognize **private context → indirect prompt injection → generated remote image/link → automatic browser or preview fetch → data in request logs**. Name the exfiltration channel and recommend controls on context access and remote-resource loading. This continues the Module 297 distinction: prompt injection influences the model, while insecure output handling turns its response into a browser/network side effect.

Notebook: [Module_307_Markdown_Exfiltration_Concepts.ipynb](Output_Attacks/Module_307_Markdown_Exfiltration_Concepts.ipynb). It shows URL encoding with fictional text only, prints a non-routable example URL, and makes no request or rendered image fetch.

## Section 7 of 14 — LLM Hallucinations

### What a hallucination is

A hallucination is generated content that is fabricated, inaccurate, nonsensical, or inconsistent, even when it sounds confident. HTB distinguishes three forms:

- **Fact-conflicting:** contradicts facts about the world (the lesson's example is incorrectly counting letters).
- **Input-conflicting:** contradicts information in the current prompt (for example, the prompt says a shirt is red but the answer says blue).
- **Context-conflicting:** contradicts earlier generated content or another statement in the same response (for example, switching from “shirt” to “hat”).

There is no single cause. Incomplete, noisy, or biased training data can contribute, as can confusing, ambiguous, or contradictory prompts. Hallucinations are inherent enough that they cannot be eliminated entirely; application design should reduce their likelihood and check important outputs before relying on them.

### Confidence signals are not proof

The section describes three ways to estimate whether a response may be unreliable:

- **Logit-based:** logits (say “LOH-jits”) are raw scores for candidate next tokens. They can be unavailable for closed models, and a high score reflects what wording the model favors—not whether the claim is true.
- **Verbalized:** ask the model to report its own confidence. A reply such as “90% confident” is a self-report, not automatically a calibrated 90% chance of correctness. Models can be poor judges of their own uncertainty.
- **Consistency-based:** prompt multiple times and compare answers. Disagreement can signal uncertainty, but repeated agreement can reproduce the same error because the runs share a model and its biases.

Multiple-agent debate can add perspectives but is not a guarantee either; agents may share assumptions or sources. Retrieval from external knowledge can ground a response in relevant material, but the source and the response still need checking. For high-impact decisions, use deterministic validation rules and human review.

### Security impact and connections

Hallucinations can spread misinformation or bias, cause financial or legal harm when users rely on fabricated policy claims, expose sensitive training data, or introduce security bugs when generated code is trusted. A fabricated package name is especially risky: an attacker may publish a malicious package under that name, and a user who installs it can run attacker-controlled code. This is a supply-chain risk, not merely a harmless install error.

This connects to earlier sections: hallucinated SQL, shell commands, or tool arguments can become dangerous when the application executes them without validation. The output-handling problem is the unsafe use of the response; hallucination describes the response being unreliable in the first place.

### COAE exam cue

Do not equate fluent or confident wording with verified truth. Recognize fact-conflicting, input-conflicting, and context-conflicting hallucinations. Treat logits, self-reported confidence, repeated answers, and agent consensus as imperfect signals. Verify claims, sources, dependencies, and code independently before consequential use.

### Notebook decision

This is a conceptual review section and introduces no safe code example that needs a notebook. The definitions and exam cues are captured here; implementation-specific mitigations are covered alongside the earlier SQL, command, tool, and exfiltration examples.

## Section 8 of 14 — Insecure Output Handling Mitigations

### Handle output for its destination

Treat every LLM response as untrusted data, just as you would user input. Apply the right control at the point it enters another interpreter or triggers an action: context-appropriate HTML encoding or safe text insertion for browser output; prepared/parameterized statements for SQL values; fixed operations and validated separate arguments for system commands; and a fixed function registry with argument validation for tools. A generic “sanitize it” step is not enough if it does not match the destination context.

### Authorization does not belong in the prompt

A system prompt that says “this function is only for administrators” is not authentication or access control. The model can ignore or be influenced around the instruction, and a prompt claim does not prove a user's identity. Enforce access in trusted application code using the authenticated user's identity, before providing data or performing an action. HTB's trust assumption is that information and functions exposed to a model should be treated as accessible through that application. In practical architecture, implement that with least privilege, per-user scoping, and only the data and tools needed for the current request; do not expose broad credentials or unrestricted backend capabilities.

### Prevent first, contain failures second

The lesson recommends sandboxing deployments that execute commands based on model output. A sandbox can restrict filesystem, network, identity, and resource access, limiting damage if prevention fails. It does not make arbitrary execution safe and should complement safer command construction, authorization, and input/output handling rather than replace them.

### COAE exam cue and notebook decision

For each scenario, identify the sink and match its defense: HTML encoding, SQL parameterization, fixed command arguments, or validated tool dispatch. Then identify who enforces authorization (trusted application code) and how least privilege limits the model's reach. Sandboxing is defense-in-depth that reduces impact; it is not the same as preventing injection. This section consolidates the patterns and notebooks from Sections 2–6, so no additional notebook is needed.

## Section 9 of 14 — Introduction to Abuse Attacks

### Deliberate misuse versus accidental error

Section 7 covered **hallucinations**: fabricated or incorrect output that can arise without a deliberate attempt to deceive. **Abuse attacks** deliberately use an LLM's text-generation or interaction capabilities to harm people or organizations. The categories can overlap: a malicious campaign may intentionally spread a false narrative, and the model may invent extra details along the way.

### Why LLMs can amplify existing harms

The section surveys four broad impacts:

- **Propaganda and psychological manipulation:** generate persuasive narratives, fake testimonials, or social-media posts at scale; automated accounts can amplify them and create the appearance of broad support.
- **Cybersecurity threats and fraud:** produce polished phishing, impersonation, payment-diversion, or other social-engineering messages. Fluency can remove some traditional spelling and grammar warning signs, so those checks are not enough to determine trustworthiness.
- **Misinformation, fake reviews, and defamation:** publish false claims or reviews quickly, misleading consumers and harming reputations before corrections catch up.
- **Hate speech:** biased training patterns or manipulative prompts can produce targeted abusive content, and scale can intensify its reach.

These are use-and-impact risks, not a specific code vulnerability. They are amplified by speed, personalization, conversational persistence, and distribution scale. Human review and platform/content policies are relevant safeguards, but should be complemented by abuse monitoring, rate limits, provenance, and reporting/escalation processes appropriate to the application.

### COAE exam cue and notebook decision

Recognize the difference between **unreliable output** (hallucination) and **deliberate harmful use** (abuse), while noting that a single incident may involve both. When describing the risk, name the harm and the scaling factors—persuasiveness, personalization, and automation. This is an overview section without new code or a practical implementation, so the study-guide notes are sufficient; no notebook is needed.

## Section 10 of 14 — LLM Abuse Attacks

### Misuse and detector evasion

The previous section surveyed harmful uses; this lesson looks at misinformation generation and attempts to evade hate-speech detectors. HTB explicitly avoids providing technical instructions for running abuse campaigns. Its misinformation example illustrates a general weakness: a model may resist a direct request about a sensitive real-world topic but produce a fictional version around a harmless placeholder that can later be repurposed. A refusal to one prompt therefore does not prove that generated content cannot be adapted or misused downstream.

### How a toxicity threshold works

A text classifier may assign a **toxicity score** and compare it with a policy cutoff. Write `flag if s ≥ τ` and say it aloud as “flag it if score s is greater than or equal to tau.” Here, `s` is the detector's score and `τ` (Greek letter tau, pronounced “tow”) is the chosen threshold. For example, if a detector scores a message at `0.82` and the cutoff is `0.70`, then `0.82 ≥ 0.70`, so the system flags it. A lower threshold tends to catch more cases but can also flag more benign content; a higher threshold may reduce false alarms but allow more harmful content through. The score reflects the classifier and its definition, not an objective measurement of harm.

Detectors can disagree because they use different definitions, training data, languages, and thresholds. The lesson's attack families change the surface form to challenge that classifier boundary:

- **Character-level changes:** swap, substitute, insert, or delete characters. The lesson uses benign misspellings to show the form of the transformation.
- **Word-level changes:** substitute a synonym, which may preserve a broad idea but can also change nuance.
- **Sentence-level changes:** paraphrase the text while attempting to retain its meaning.

The general red-team question is whether content with the same harmful intent can receive a lower detector score after its form changes. This can also affect detectors for other policy categories. Detection should be evaluated on diverse variations and with human review where decisions have significant consequences.

### COAE exam cue and notebook decision

Distinguish **harmful content generation** from **evasion of a harmful-content detector**. For a threshold question, identify the score, the cutoff, and which side triggers the flag; remember that the threshold trades off false positives and false negatives. The evasion goal is to retain meaning while changing form enough to cross the classifier's decision boundary. This section is conceptual and intentionally does not need a notebook containing harmful content or evasion payloads.

## Section 11 of 14 — Mitigating Abuse Attacks

### Defense in layers

HTB groups mitigation into model safeguards, monitoring, and public awareness:

- **Before deployment:** adversarial training exposes the model to challenging prompts during training to improve resilience; adversarial testing probes a model to find weaknesses before release. Bias detection can help identify problematic patterns in training data.
- **At deployment:** context-aware guardrails and content filtering/moderation can block harmful requests or outputs. These are runtime layers and can still miss cases, so do not make them the only control.
- **Monitoring and verification:** AI-generated-text detection estimates likely authorship; misinformation detection and fact-checking evaluate whether claims are supported by reliable sources. Authorship and truth are different properties.
- **Watermarking:** a generator can slightly adjust token-selection probabilities in a detectable pattern. Later statistical analysis may provide evidence about a text's origin. It aims to leave a subtle trace, not to verify a claim's truth or classify its safety.
- **Policies and public literacy:** legal/industry rules, media literacy, awareness campaigns, and critical thinking can reduce the spread and impact of harmful content.

### Practical limits and COAE exam cue

Layer controls across the lifecycle: **training/testing → runtime safeguards → monitoring/fact-checking → policy and public awareness**. No single layer is sufficient. As a practical caution beyond the lesson's overview, AI-text detectors can make false-positive and false-negative errors, and watermark evidence may be unavailable or weakened by editing/paraphrasing. Treat these signals as evidence to investigate, not definitive proof. Verify factual claims against trustworthy sources and keep human review for consequential decisions.

This section is conceptual and does not introduce an implementation that benefits from a notebook. The study-guide entry captures HTB's defense categories and the limitations important for certification scenarios.

## Section 12 of 14 — Safeguard Case Studies

### Compare the safeguards by purpose

HTB compares three safeguards that operate at different points and cover different risks:

| Safeguard | How the section presents it | Main detection focus in this lesson |
| --- | --- | --- |
| **Model Armor** | A managed service integrated as a sanitization/inspection layer before and after the primary LLM. The app submits a prompt, checks the result, sends it to the LLM, then checks the response before returning it. | Prompt attacks and abuse content such as hate speech/harassment; HTB says it does not detect misinformation. |
| **ShieldGemma** | A separately integrated, fine-tuned model queried on prompt or response text under an explicit safety policy, often expecting a Yes/No classification. | Hate speech and harassment. Its evaluation prompt and expected format matter. |
| **Prompt Guard** | A safeguard focused on prompt attacks. | Prompt injection and jailbreaking; it is not presented here as an abuse-content or misinformation detector. |

### Read the API example without running it

The Model Armor example uses configuration values such as `LOCATION`, `PROJECT_ID`, and `TEMPLATE_ID`, plus an `AUTH_TOKEN`. The request sends JSON prompt text to a REST endpoint and authenticates with a bearer token. The operation name `sanitizeUserPrompt` indicates that this is the **input-inspection** stage; the section also describes inspecting the generated response before serving it. `MATCH_FOUND` means the configured filter found a match, and a label such as `MEDIUM_AND_ABOVE` is a confidence category returned by the service—not a guarantee of complete detection. HTB explicitly says a cloud account is not needed to follow the lesson, so no credentials or API call are needed for this study guide.

### COAE exam cue and notebook decision

Identify **where** a safeguard sits (before the model, after it, or both) and **what** it is designed to detect. Do not treat any one safeguard as universal: in this lesson, Model Armor covers prompt attacks and abuse but not misinformation; ShieldGemma focuses on hate speech/harassment; Prompt Guard focuses on prompt injection/jailbreaking. Keep API credentials out of notebooks and source control. Product features can change, so use current vendor documentation for deployment decisions; this table records the distinctions HTB teaches in this section. No notebook is added because the code is an optional, credentialed vendor API call rather than a safe offline example.

## Section 13 of 14 — Legislative Regulation

### Why regulation is difficult

Legislation tries to reduce harms such as fraud, non-consensual deepfakes, and abusive campaigns while preserving useful innovation and fundamental rights such as freedom of expression. It also has to decide which actors have duties: the model provider, the organization deploying it, the platform distributing the content, or the user who prompted it. Those roles and obligations depend on the law and facts; a technical safeguard alone does not settle legal responsibility.

### United States examples in the lesson

HTB contrasts broad protection for speech in the U.S. with other legal rules that can apply to specific conduct, such as fraud, defamation, or incitement. It highlights the **TAKE IT DOWN Act**, which includes realistic AI-generated or altered intimate depictions of identifiable people in certain circumstances and requires covered platforms to create a notice-and-removal process. The Act became Public Law 119-12 on May 19, 2025; its criminal prohibition and platform process have separate effective dates. This is a targeted example, not a general ban on misinformation.

HTB also mentions the **NIST AI Risk Management Framework (AI RMF)** and the **FTC**. The AI RMF is voluntary risk-management guidance for organizations across the AI lifecycle. FTC consumer-protection authority may apply to deceptive or fraudulent commercial practices involving AI; this does not make the FTC a general speech-content regulator.

### European Union examples in the lesson

The **Digital Services Act (DSA)** sets obligations for digital services, including user reporting mechanisms and responses for illegal content. The European Commission clarifies that the DSA does not itself decide what content is illegal; that definition comes from other EU or national law. Extra systemic-risk assessment and audit/reporting duties apply to designated very large online platforms and search engines (VLOPs/VLOSEs), rather than identically to every service.

The **EU AI Act** is a separate, AI-specific risk-based framework. HTB introduces prohibited, high-risk, limited-risk, and minimal-risk categories. An important real-world nuance is that classification depends on an AI system's intended purpose and use, not simply the fact that it is an LLM. For example, an LLM used in a high-risk context may have different obligations from a general chatbot. The Commission's current materials list transparency duties for certain AI interactions and generated content; schedules differ by obligation and are subject to legislative updates.

### COAE exam cue, current-reference caveat, and notebook decision

Distinguish **voluntary guidance** (NIST AI RMF) from **binding legal obligations** (for example, the TAKE IT DOWN Act, DSA, and AI Act). Know the broad contrast HTB teaches: U.S. protections and targeted laws versus the EU's platform and risk-based AI frameworks. Do not overgeneralize that all misinformation is illegal, that the DSA defines illegality, or that every LLM automatically has the same EU risk category. Because this area changes, use the official materials below for current scope and dates when applying the concepts outside the course. This is a conceptual section with no code, so no notebook is needed.

### Official references for current detail

- [TAKE IT DOWN Act, Public Law 119-12](https://www.congress.gov/119/plaws/publ12/PLAW-119publ12.pdf)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [European Commission: Digital Services Act Q&A](https://digital-strategy.ec.europa.eu/en/faqs/digital-services-act-questions-and-answers)
- [European Commission: DSA obligations for very large platforms and search engines](https://digital-strategy.ec.europa.eu/en/policies/dsa-vlops)
- [European Commission: AI Act risk-based framework](https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai)

## Section 14 of 14 — Skills Assessment: LLMPics

### Scenario and scope

The assessment asks for a security assessment of the spawned `LLMPics` website and the lab flag. HTB provides a test account in the section and says to use the web application directly in a browser; this assessment does not require SSH port forwarding. The spawned host and port are temporary, so enter the current instance into the notebook rather than committing it.

Initial browser reconnaissance found an `Imagebot` that can return a random photo or search for an image, a `(Beta) Adminbot` route that asks for an `admin_key`, and a login/profile flow. An unauthenticated request to Adminbot returned “Please provide your admin_key”; a deliberately incorrect test value returned “Invalid admin_key.” A benign Imagebot question identified its supported photo-search behavior. A direct request to reveal hidden configuration was refused. Some follow-up Imagebot submissions produced no visible result; treat those as inconclusive rather than proof of a vulnerability or a fix. These observations do not establish the full backend design or the route to the flag.

### Assessment method

Proceed from the visible application and test one hypothesis at a time. Use a benign baseline, observe what the bot returns and what the app does with it, then choose a technique based on evidence. The module's relevant boundaries include generated output rendered in HTML/Markdown, database search, command execution, tool/function dispatch, and access to data or an admin capability. Distinguish a model claim from a verified application effect. Do not brute-force credentials or keys; keep the work scoped to the current spawned instance.

The accompanying notebook holds an empty target host/port, provides a narrowly scoped read-only page inspector, and records only redacted observations. It does not store the test password, a recovered key, or the flag. The assessment remains in progress until the Academy confirms completion.

### COAE exam cue

For a black-box LLM web app, map user-controlled input → model/tool → output sink or side effect → access-control check. Identify whether the failure is prompt injection, insecure output handling, excessive agency, or a vulnerable downstream function; these can chain together. Verify impact at the application boundary and recommend the corresponding control, not just a stronger prompt.

Notebook: [Module_307_LLMPics_Skills_Assessment.ipynb](Skills_Assessment/Module_307_LLMPics_Skills_Assessment.ipynb).
