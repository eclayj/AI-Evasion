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
