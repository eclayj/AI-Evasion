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
