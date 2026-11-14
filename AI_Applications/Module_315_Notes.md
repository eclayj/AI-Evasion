# Attacking AI — Application and System

Source: HTB Academy, [Attacking AI - Application and System](https://academy.hackthebox.com/app/module/315). These notes are organized by the module's sections for COAE study. Review sections stay concise; new techniques, code, and exercises receive fuller walkthroughs. A notebook will be added when a section has useful code to study or run.

## Section 1 — Overview of Application & System Components

This opening section shifts the security lens from the model and its training data to the software and infrastructure around them. A deployed AI service is a connected system with four broad components:

| Component | Plain-language role | Typical security question |
| --- | --- | --- |
| **Model** | Turns inputs into predictions or generated output | Can inputs manipulate behavior, or can outputs reveal learned information? |
| **Data** | Supplies training, reference, or user information | Can records, labels, or sensitive examples be changed or exposed? |
| **Application** | Connects users to the model and its tools: web/mobile interfaces, APIs, databases, plugins, and agents | Can a request bypass authorization, inject instructions into another interpreter, or trigger an unsafe tool action? |
| **System** | Runs and stores the application: source code, model/data files, cloud or local infrastructure, networks, hardware, and deployment pipeline | Can the service, artifact, secret, or deployment process be reached or changed by an unauthorized party? |

A useful mental picture is **model/data are what the AI works with; application/system are how people and other services reach it and where it runs**. These categories interact. For example, an application flaw may expose training data, and a compromised deployment pipeline may replace model files. A secure model does not compensate for an application that gives it unsafe tools or a server that exposes its artifacts.

### Application-component risks

The application is the interface layer. It may combine ordinary web software with AI-specific features, which means classic application vulnerabilities can affect the AI deployment too.

- **Injection:** SQL or command injection occurs when untrusted input is treated as part of a command or query. Depending on the vulnerable path, an attacker may read or alter records, run commands, or disrupt the service.
- **Access-control failure:** a user can reach a record, tool, or operation that their identity should not permit. Authentication asks **who are you?** Authorization asks **what may you do?** A valid login does not establish permission for every object or action.
- **Denial of ML service (DoML):** requests consume enough model time, memory, or paid inference capacity to impair availability. A normal-looking API request can still be abusive if it is too frequent or expensive.
- **Rogue actions and excessive agency:** an application gives a model tools or permissions beyond the task's needs. The model may then perform a harmful action due to attacker-influenced input or an unexpected interaction. The underlying control failure is often excessive capability or missing human/logic checks.
- **Model reverse engineering / extraction:** repeated input-output queries can reveal enough behavior to approximate a proprietary model. Rate limits and monitoring constrain query volume; they do not prove that a model is impossible to copy.
- **Vulnerable agents or plugins:** an integrated component may contain its own flaws, misuse its permissions, or send interaction data somewhere unintended. Every integration adds another trust boundary.
- **Sensitive-data logging:** prompts, responses, credentials, or personal data can leak through logs even when the model itself behaves correctly. Logs need data minimization, access control, and an appropriate retention period.

### System-component risks

The system component includes the infrastructure and processes that build, deploy, and operate the service. A weakness here can affect the entire deployment rather than one model response.

- **Misconfigured infrastructure** can expose storage, management interfaces, model files, datasets, or secrets to the public.
- **Poor patch management** leaves known vulnerabilities in the operating system, application server, libraries, or ML stack. The impact can range from privilege escalation to remote code execution, depending on the flaw.
- **Weak network security** makes it easier to reach internal services or move between systems. Segmentation limits which systems can talk; encryption protects traffic in transit; monitoring helps identify suspicious use.
- **Deployment tampering** can alter source code, model files, or release steps, changing the behavior of the running AI service. This connects application/system security to software and model supply-chain integrity.
- **Excessive data handling** means collecting or retaining more user data than the service needs. That increases exposure if storage is breached and may create privacy or legal problems.

### Model Context Protocol (MCP): first orientation

HTB introduces the **Model Context Protocol (MCP)** as a standardized interface for an LLM application to connect with external resources and services. Before a common protocol, each application often needed a separate connector for each service. A shared protocol can simplify those connections, but it does not make a connected server trustworthy by itself.

For security review, treat every MCP-connected server or tool as part of the application's trust boundary. Ask what data it can read, what actions it can perform, what identity it uses, and whether the model or user can cause those capabilities to be invoked. This is only the overview; later module sections examine MCP's operation and attack surface.

### COAE quick recall

- Classify the weakness by **component**: model, data, application, or system. One incident can cross several components.
- **Authentication** verifies identity; **authorization** checks permission for a specific object or action.
- DoML primarily threatens **availability**; sensitive logs or model extraction threaten **confidentiality**; deployment tampering threatens **integrity**.
- Excessive agency is a capability/permission problem: give agents only the tools and scope they need, and gate sensitive actions.
- MCP standardizes connections; security still depends on each server's trustworthiness, permissions, and data handling.
- For any scenario, identify the entry point, the component, the trust boundary, the capability involved, the affected security property, and a practical control.

**Exam check:** If an AI assistant can issue database queries, identify one application risk and one control that limits damage. If model files are replaced during release, which component and security property are affected? Why does adopting MCP not automatically make its connected servers safe?

**Notebook decision:** This section is conceptual and introduces no runnable code. The annotated module notebook begins in Section 2, where HTB demonstrates black-box model extraction.

## Section 2 — Model Reverse Engineering

### What the attack does

**Model reverse engineering**, also called **model extraction** or **model stealing**, uses an exposed prediction interface as a source of labels. The attacker chooses inputs, asks the original model for its answers, and trains a separate **surrogate model** to imitate those answers. It is a black-box attack: the attacker need not see the original architecture or its training dataset.

In HTB's example, the input has two numeric features—penguin flipper length and body mass—and the API returns a class label (`Adelie` or `Gentoo`). The attack proceeds by generating plausible feature pairs, querying the API for labels, and fitting logistic regression to the collected input/output pairs. The surrogate can learn a similar decision boundary even if its internal design differs from the original model.

### Read the notation aloud

- `xᵢ` is **“x sub i”**: the i-th input row, such as one penguin's two measurements.
- `F(xᵢ)` is **“F of x sub i”**: the original model's prediction for that row.
- `gθ(xᵢ)` is **“g theta of x sub i”**: the surrogate's prediction. Theta (“theta”) stands for the learned settings of the surrogate.
- `θ* = arg minθ Σᵢ loss(gθ(xᵢ), F(xᵢ))` is read **“theta star equals the value of theta that minimizes the sum over i of the loss between g theta of x sub i and F of x sub i.”** Here, `Σᵢ` (“sum over i”) adds the errors over queried examples, and `arg min` means “the setting that makes the expression as small as possible.”

In ordinary English, fit the surrogate's settings so its answers agree with the original model's answers on the collected examples. For a tiny example, if the original returns `Adelie, Gentoo, Gentoo` for three sampled points, training adjusts the surrogate so it learns those same labels and the pattern separating the classes.

### Sampling and evaluation

HTB samples flipper lengths from 150–250 mm and body masses from 2,500–6,500 g. `Uniform(150, 250)` is read **“uniform from 150 to 250”**: the sampler draws values across that interval with equal density. Restricting queries to realistic ranges gives the surrogate more useful examples than sampling arbitrary values. The section uses 100 examples for a deliberately simple demonstration and reports roughly 98% accuracy on the lab's test set.

Accuracy can be written `1/n × Σᵢ 1[gθ(xᵢ) = F(xᵢ)]`. Read this as **“one over n times the sum over i of the indicator that the surrogate prediction equals the original prediction.”** The indicator is 1 when the predictions match and 0 otherwise. If the models agree on 98 out of 100 test examples, their agreement accuracy is `98/100 = 0.98`, or 98%.

The lab endpoint evaluates the submitted surrogate against its own test data. In production evaluation, keep evaluation examples separate from surrogate training examples; high agreement on training queries alone can overstate how well the copy generalizes.

### Keep related attack goals distinct

- **Extraction** aims to reproduce model behavior or steal intellectual property.
- **Inversion** aims to infer sensitive information about training data. A copied model may help an inversion attempt, but extraction and inversion are different objectives.
- A high agreement score says the copy behaves similarly on the evaluated distribution. It does not prove identical internals or guarantee similar behavior on every possible input.

### Defenses and exam cues

Rate limits and monitoring make bulk querying slower, more visible, or more expensive. Other controls include limiting unnecessary output detail, detecting repeated or systematically varied queries, and protecting the model endpoint with appropriate access controls. These controls have tradeoffs: overly strict limits can degrade legitimate service, and no single rate limit proves extraction is impossible.

For a scenario question, identify the exposed prediction interface, what information each response reveals, whether the attacker's inputs cover the useful domain, and how similarity is evaluated. If the attacker trains a copy from queried labels, call it **model extraction**. If the goal is recovering a sensitive training example or attribute, call it **model inversion**.

**Notebook:** `Module_315_Model_Extraction_Study.ipynb` follows HTB's sampling → API query → surrogate training → lab evaluation flow. The target URL is configurable, requests are bounded and timed out, and the final model upload is disabled by default. The notebook is an educational reproduction of the lab, not a production extraction tool.

## Section 3 — Denial of ML Service

### What makes this ML-specific

Denial of ML service (DoML) threatens **availability** by consuming resources needed for inference. A conventional DoS may flood a service with traffic; an ML-specific DoS can use requests that look valid but are expensive for the model to process. Impact can include slower responses, outages, depleted battery, or increased cloud cost. Inputs can target worst-case algorithmic behavior even when the input size is not unusually large.

### Sponge examples and notation

A **sponge example** is crafted to increase inference latency or energy use without increasing input dimension. For text models, two factors can increase work: a large output token count and a large input token count. Character length is not token count. Common phrases may map efficiently to a few tokens; rare words or unusual character sequences may split into many tokens.

The search objective can be summarized as `x* = arg max latency(x), subject to input_length(x) ≤ 50`. Read it aloud: **“x star is the input, among inputs whose length is at most 50, that has the greatest latency.”** `arg max` means “the input that maximizes”; “subject to” introduces the length limit. The attacker searches for costly inputs while holding size within a bound.

HTB introduces a genetic algorithm as a way to search candidate inputs: initialize a population, measure fitness (latency or energy), select high-cost candidates, combine them with **crossover**, apply small random changes with **mutation**, replace the population, and repeat. In a black-box setting, the attacker needs only the service and a way to measure response time. In a white-box setting, model architecture and parameters are available. The section describes the attack concept; our notebook does not generate sponge examples or send load to a service.

### Reading the reported measurements

In the cited GPU experiment, natural inputs used 9,492 mJ and 0.10 ms; random inputs used 25,773 mJ and 0.24 ms; sponge inputs used 40,976 mJ and 0.37 ms. Thus, for that experiment, sponge energy was about `40,976 / 9,492 ≈ 4.3` times natural energy, and latency was `0.37 / 0.10 = 3.7` times natural latency. A separate black-box Azure result reports about 1 ms for ordinary translation and about 6 seconds for a sponge input capped at 50 characters: 6 seconds is 6,000 ms, or approximately `6,000 / 1 = 6,000` times as long. These are research results for specific tasks and deployments, not universal performance guarantees.

### Defenses and COAE recall

Rate limits and anomaly detection remain useful, but defenses should also constrain maximum input length, token budget, output length, concurrency, and per-request runtime or energy. Systems can stop a request that exceeds a configured resource threshold. Thresholds and rate limits need tuning: set them too low and legitimate long or complex requests fail; set them too high and resource exhaustion remains possible. Robust model and service design should handle unusual but valid inputs without severe slowdowns.

For an exam scenario, identify the affected property (**availability**), resource being exhausted (compute, energy, latency budget, or paid inference), whether the attacker needs internal access or only query timing, and a layered mitigation that preserves ordinary use.

**Notebook:** The existing model-extraction notebook is extended with an offline tokenizer comparison and calculations for the reported measurements. It does not send sponge inputs to any service or generate attack candidates.

## Section 4 — Insecure Integrated Components

### Follow the data and identity across components

An AI application may combine a web front end, a database, an LLM, and plugins. A weakness in any connector can expose conversations or let an action occur under the wrong authority. Pixel Forge shows two routes to stored conversations: a normal web endpoint and a chatbot plugin that summarizes conversations. The web route may enforce ownership correctly while the plugin does not; security must be checked independently at every route.

### Web application: IDOR and SQL injection

The route `/query/5` uses an integer as a direct reference to one saved conversation. **IDOR** is pronounced **“eye-door”** and means *Insecure Direct Object Reference*: the application accepts a record ID without verifying that the signed-in user may access that record. HTB tests IDs from 1 to 100 with `ffuf`, a web fuzzer that substitutes candidate values into a URL. The example returns only ID 5 for the current user, suggesting that this route checks ownership. A status code alone is not enough; inspect which record and owner the response represents.

The page then appends a quote to the ID and receives a SQL syntax error. This is a clue that the input may be inserted into a database query unsafely. A `UNION SELECT` payload combines rows from another query with the application's result. Its number of selected columns must match the original query. HTB's illustrative three-column payload is:

```text
/query/x' UNION SELECT 1,2,3 -- -
```

Read it as **“query slash x, close the current quoted value, union-select one-two-three, then comment out the rest.”** Here `1,2,3` are placeholder values for three result columns, and `-- -` begins a SQL comment in the lab's database dialect. The application displays values from the injected query, confirming that database input is being interpreted as SQL. In the lab this can expose stored chatbot interactions. In secure code, use parameterized SQL so an ID is always treated as data, and separately check that the authenticated user owns the requested record.

**IDOR and SQL injection are different:** IDOR is a missing authorization check for a valid object reference. SQL injection is a failure to separate data from executable query syntax. One endpoint may have both kinds of risk, so test and fix them independently.

### Plugin: authorization must not depend on the model

The chatbot exposes functions such as checking an order or summarizing a conversation. A plugin that accepts both a conversation ID and a user ID must not trust the LLM to choose the user ID. If that value comes from model-generated text, prompt injection may influence it. The plugin can then act as a **confused deputy**: a component with legitimate database authority is tricked into using it on the wrong user's behalf.

The stronger design is to derive the user's identity from the server's authenticated session, pass that trusted identity into the plugin, and perform the conversation-ownership check in the server/plugin/data-access layer. Treat model output as untrusted input. A chatbot refusal is useful behavior, but it is not an authorization boundary.

### HTB lab command patterns and handling credentials

HTB's enumeration example uses `seq` to generate candidate IDs and `ffuf` to substitute each one for `FUZZ`. A generic version is shown below; use only the active authorized lab host and your own session cookie. **Never put a real cookie in notes, a committed notebook, a screenshot, or a command history that may be shared.**

```bash
seq 1 100 | ffuf -u 'http://<SERVER_IP>:<PORT>/query/FUZZ' -w - -mc 200 \
  -b 'session=<YOUR_SESSION_COOKIE>'
```

The SQL tests shown by the course are likewise scoped to the lab endpoint. A quote that produces a database error is an indicator to investigate, not proof by itself; confirm only within the authorized environment and stop when the intended learning objective is met.

### Mitigations and COAE recall

- Enforce object-level authorization for every request, including requests made through plugins. Derive identity from the authenticated server context, not from a user-controlled or model-generated parameter.
- Use parameterized queries, validate types and ranges, and return safe errors that do not expose database details.
- Apply least privilege to plugins and tools. Give each integration only the records and actions it needs; review its origin, code, dependencies, and necessity.
- Treat user input and model output as untrusted at each processing boundary. Validate tool arguments and results, constrain side effects, and require human approval for sensitive actions when appropriate.
- Use defense in depth: access controls, sanitization, rate limits, monitoring, logging with care, and sandboxing. Logging itself must not expose conversation contents or secrets to unauthorized readers.

For an exam scenario, map the path from user to web route or LLM to plugin to database. Ask where the ID came from, which identity the system trusts, whether the SQL query treats input as data, and whether object ownership is checked at the final data access. Name both the weakness and a control at the layer where it belongs.

**Notebook decision:** No new Python implementation is needed here. The hands-on portion is an authenticated web application lab using the browser and `ffuf`; the study notes keep copyable lab patterns with placeholders and do not store session credentials.

## Section 5 — Rogue Actions

### Rogue actions and excessive agency

A **rogue action** is an unintended or harmful operation performed by an AI application, often through a plugin, agent, or extension. It can happen because the model misinterprets normal input or because an attacker deliberately exploits it. **Excessive agency** means the system gives the model too much freedom or too many capabilities without sufficient limits or human oversight. Agency describes the power available; a rogue action is one possible result.

Pixel Forge's chatbot has an `OrderStatus` plugin, a `ConversationSummary` plugin, and an administrator-only `SQLQuery` plugin. The SQL tool can issue database queries, so the risk includes loss of confidentiality (reading data) and integrity (changing or deleting data). HTB demonstrates successful tool invocation with a read-only query that returns the database version.

### Direct prompt injection

The chatbot initially refuses to use `SQLQuery` because the user is not an administrator. HTB then shows that a direct prompt injection—claiming administrator status and instructing the model to use the plugin—can bypass a restriction that exists only in the model's behavior. This is an authorization design flaw: **the LLM must not decide who is an administrator**. The application must enforce that permission at the tool boundary on every call.

### Indirect prompt injection: untrusted data reaches a privileged assistant

An attacker may be unable to access the administrator chatbot directly. The indirect path works through content the attacker controls:

1. The attacker registers a username containing instruction-like text.
2. They place an order, creating a normal order record.
3. An administrator asks an assistant to check the order; the `OrderStatus` plugin returns both the order status and username.
4. The privileged assistant reads the attacker-controlled username as part of its context.
5. The embedded instructions may persuade it to invoke a tool that is available in the administrator's context.

This is **indirect prompt injection**: the attack instruction arrives inside data the model retrieves or processes, rather than as a direct instruction in the current user's message. It creates a confused-deputy risk—the assistant has legitimate access, but attacker-controlled content tries to steer how that access is used. Even when a scenario starts with a low-privilege user, retrieved content can cross into a more privileged workflow.

The example query only reads the database version. The same flaw could be more severe if the model can run arbitrary SQL or perform write operations. A username, product description, document, or email should remain untrusted data even after a trusted tool retrieves it.

### Defenses and COAE scenario analysis

- Enforce administrator and user permissions in application code at each tool call; do not rely on a prompt, system message, model refusal, or user-supplied role claim.
- Remove general-purpose tools where a narrow capability is sufficient. Prefer a validated `get_order_status(order_id)` function over arbitrary `SQLQuery` access.
- Apply least privilege and per-user scoping. A tool should receive only the identity and data its task requires; derive identity from the authenticated server context.
- Separate untrusted retrieved content from trusted instructions, label and handle it as data, and validate tool arguments and outputs. Prompt wording alone cannot guarantee that injection will fail.
- Require a person to approve high-impact actions such as deleting data, sending messages, making purchases, or modifying production systems. Log tool invocations and monitor for unusual actions without retaining unnecessary sensitive content.
- Use sandboxing, read-only access, query allowlists, and execution limits to reduce impact if the model is manipulated.

For a COAE scenario, name the attacker-controlled content, the assistant's privilege, the tool capability, the requested side effect, and the control that must enforce authorization independently of the LLM. Distinguish **direct** injection (instructions arrive in the user's prompt) from **indirect** injection (instructions arrive through retrieved or processed content). Identify whether the impact is confidentiality, integrity, availability, or a combination.

**Notebook decision:** This section is an interactive chatbot/tool-permission lab with no new Python implementation. The notes capture the direct and indirect attack chains and their controls; they do not automate privileged plugin calls.

## Section 6 — Excessive Data Handling & Insecure Storage

### Minimize what the AI application collects

**Excessive data handling** means collecting, processing, or retaining information beyond what the application's purpose requires. In the Pixel Forge example, the chatbot asks about a medical condition to recommend a console and then asks for payment details in a chat. A recommendation feature may need a broad preference, but it generally does not need a diagnosis or a card number. Sensitive information entered into a chat may also be copied into prompts, logs, analytics, backups, or training pipelines.

The **principle of data minimization** is: collect only the fields needed for the task, explain why they are needed, and avoid retaining them longer than necessary. Minimization reduces both the chance of exposure and the amount of damage if a system is breached. A reasonable design routes payment through a dedicated payment provider and keeps card details out of general chatbot conversations and logs.

### Insecure storage and the exposed database example

The lab uses `gobuster` to enumerate likely web paths and filenames. A wordlist supplies candidates; `-x .db,.txt,.html` asks it to try those suffixes too. The lab discovers a web-accessible `storage.db` file. The course then inspects the file and finds an `llm_queries` table containing user IDs, IP addresses, prompts, and responses. The example demonstrates how logs can expose sensitive details that the main application did not need to keep.

An HTTP `200` response means that the server returned content; it does not prove the content is intended to be public. In an authorized assessment, verify the response and its access requirements. Do not copy real customer records or credentials into notes. Other places to examine include API responses, application logs, object storage, caches, backups, and data-processing pipelines.

The course's lab command pattern is:

```bash
gobuster dir -u 'http://<SERVER_IP>:<PORT>/' \
  -w /opt/useful/seclists/Discovery/Web-Content/raft-small-words.txt \
  -x .db,.txt,.html
```

In the lesson, the discovered file is then retrieved from the lab host for inspection. Keep that action confined to an authorized lab; in a real assessment, first confirm scope and avoid downloading sensitive records when metadata or a redacted sample can establish the finding.

### Storage controls and privacy terms

- **Access control:** authenticate users and authorize each read or write, including direct requests to files and APIs. A hard-to-guess filename is not an access control.
- **Encryption:** protect data in transit and at rest. Encryption is useful, but it does not repair an application that is authorized to expose every record to the wrong user.
- **Retention:** define when records expire and delete them when the business purpose ends. Include logs, backups, and derived datasets in the policy.
- **Anonymization:** transform data to reduce the ability to identify people. Removing a name alone may not be enough when other fields can be combined to re-identify someone.
- **Differential privacy:** add calibrated randomness to a supported statistical or learning process so its output reveals limited evidence about any one individual's inclusion. It can protect aggregate outputs; it does not automatically secure raw chat logs.
- **Homomorphic encryption (HE):** permits certain computations to be performed on encrypted data, with results that remain encrypted until decrypted. This can provide confidentiality during computation, but often adds substantial performance cost and does not replace authorization or data minimization.

### COAE scenario recall

When a scenario describes sensitive AI inputs being logged or exposed, separate the failure into four questions: **Was the data necessary? Who can access it? How is it protected? How long is it retained?** Match controls to the failure: minimize collection, enforce server-side access checks, encrypt data appropriately, and set deletion schedules. Use privacy-preserving techniques such as differential privacy or HE only where their threat model and workload fit.

**HTB's lesson:** enumerate the lab web application, notice the exposed database file, and inspect what kinds of LLM data were logged.

**Production perspective:** use explicit assessment scope and synthetic or redacted records; avoid retrieving real medical or payment data merely to prove exposure. A production review can often establish the problem through access rules, configuration, schemas, and carefully limited evidence.

**Notebook decision:** No new Python notebook is needed. This is a web-application discovery and data-governance lab; a Python implementation would add little learning value. The existing model-extraction notebook does not need changes for this section.

## Section 7 — Model Deployment Tampering

### Ways a deployed model can be changed

**Direct model tampering** is unauthorized access to the deployed model artifact followed by changes to its parameters, configuration, or files. For a neural network, **weights** and **biases** are learned numeric values that influence its output. A small or targeted alteration can change decisions while leaving most normal behavior intact.

**Indirect model tampering** changes an upstream input to the model-building process, such as its training data. **Data poisoning** means deliberately inserting or modifying examples so that training produces attacker-desired behavior. A **backdoor** is a hidden behavior that activates for a particular trigger but remains hard to notice on ordinary inputs. For example, a classifier could behave normally except when an otherwise irrelevant trigger appears.

The distinction to remember is where the change enters: direct tampering changes the deployed model or its deployment files; indirect tampering changes training data or another upstream pipeline input. Both can harm model integrity and produce similar downstream behavior.

### ShellTorch: how separate weaknesses compose

HTB presents **ShellTorch** as an example of a model-serving system compromised through a chain of infrastructure weaknesses. TorchServe hosts models and exposes management functions. The lesson's chain is:

1. A management API is reachable without the expected authentication or network restrictions.
2. A model-loading feature accepts a URL and causes the server to fetch remote content. This is **SSRF**, pronounced **“ess-ess-are-eff”**: *Server-Side Request Forgery*. The attacker influences a request made from the server's network position.
3. The fetched content is processed by a vulnerable YAML deserialization path. **Deserialization** (say **“dee-sir-ee-uh-luh-zay-shun”**) rebuilds program objects from structured data. Unsafe object reconstruction can trigger behavior while the data is being loaded.
4. The resulting object chain reaches code execution on the server. **RCE**, pronounced **“R-C-E,”** means *Remote Code Execution*.

This chain is valuable for security reasoning because no single label describes the whole failure. Weak access controls expose a powerful API; SSRF lets the server retrieve attacker-controlled content; unsafe deserialization processes that content dangerously; and the final impact is RCE. In a COAE scenario, state both each link and the end impact instead of stopping at “the model was hacked.”

### Defenses across the deployment lifecycle

- Restrict management interfaces to trusted networks and authenticated administrators; do not expose them publicly by default.
- Validate and constrain server-side URL fetching. Use allowlists where possible, block internal and metadata destinations, and apply network egress controls.
- Avoid unsafe deserialization of untrusted input. Prefer simple data formats and parsers that do not instantiate arbitrary program objects.
- Patch model servers, YAML libraries, container runtimes, and other dependencies. Track versions and respond to security advisories.
- Run serving processes with least privilege, isolate them, and limit their filesystem and network access so a service compromise has less reach.
- Protect model and training-data storage with access controls, versioning, integrity verification, and audit trails. Use controlled, reviewable build and deployment pipelines and verify model artifacts before promotion.
- Test model behavior and pipeline inputs, including targeted trigger cases, while recognizing that ordinary accuracy tests may not reveal a carefully hidden backdoor.

**HTB's lesson:** the lab demonstrates that weaknesses in deployment infrastructure can combine to compromise a model-serving server and put models and data at risk.

**Production perspective:** the course's exploit chain is specific to its authorized lab setup and vulnerable software versions. Do not run exploit payloads against real model servers without explicit scope. A production assessment can begin with version/configuration review, access-control checks, dependency scanning, and safe verification of network boundaries.

**Notebook decision:** No new Python notebook is needed. The section's hands-on chain uses SSH tunneling, a vulnerable model-serving API, unsafe YAML deserialization, Java payload construction, and remote code execution. Packaging those steps into a reusable notebook would make an environment-specific exploit too easy to aim at the wrong system. The existing Module 315 notebook remains focused on safe model-extraction concepts.

## Section 8 — Vulnerable Framework Code

### Framework flaws affect the AI system around the model

ML applications depend on frameworks for serving models, tracking experiments, downloading artifacts, and handling data. A flaw in one of those components can affect availability or expose server files even if the model itself is behaving as designed. These are also **supply-chain risks**: the application inherits weaknesses from software components it depends on.

### Ollama: a denial-of-service example

HTB describes **CVE-2025-1975** in Ollama version **0.5.11**. A size-checking error while processing a downloaded model manifest could cause an out-of-range operation and crash the server. This is a **DoS** (pronounced **“D-O-S,”** *Denial of Service*) vulnerability: the demonstrated impact is to make the service unavailable, so the primary security property affected is **availability**.

The lesson is that manifests and other model-download metadata are still untrusted input. Validate their structure and lengths before using them, and keep the affected framework patched. A crash during parsing is a security issue even if no data is stolen and no code is executed.

### MLflow: runs, experiments, and LFI

**MLflow** tracks work across the ML lifecycle. A **run** is one execution of training or another job, with its recorded parameters and outputs. An **experiment** groups related runs so a team can compare them—for example, several attempts to train the same classifier with different settings.

HTB then covers **CVE-2023-6909** in MLflow **2.7.1** and the related **CVE-2024-1594** in **2.9.2**. These are **LFI** (pronounced **“L-F-I,”** *Local File Inclusion*) issues. A URL-to-path handling weakness let a user-controlled artifact location escape its intended directory and retrieve other files from the tracking server.

The path marker `../` is read **“dot-dot-slash”** and means “go up one directory.” Repeating it can traverse toward a parent directory or filesystem root if the application does not correctly constrain the resolved path. In these examples, the first fix checked for traversal in a URL query string. A follow-up issue used a URL fragment instead, showing why checking just one textual part of a URL is not robust validation. URL parsers and application frameworks can interpret strings differently; validation should operate on a parsed, normalized value and enforce the final allowed filesystem boundary.

The primary impact of LFI is **confidentiality**: files may reveal configuration or other sensitive information. Exposed secrets may then enable further access. In COAE scenario questions, distinguish this file disclosure from the Ollama DoS: one affects confidentiality; the other affects availability.

### COAE recall and controls

- Record the affected component and version, the attacker-controlled input, and the operation that processes it.
- Classify the direct impact: a crash or resource exhaustion is DoS/availability; reading files outside the intended directory is LFI/confidentiality.
- Patch vulnerable framework versions and track dependencies across model-serving and experiment-tracking systems.
- Keep management and tracking services behind authentication and network restrictions; do not expose them broadly just because they are “internal ML tooling.”
- Validate manifests and structured data, check sizes and types, and use safe parsers.
- Resolve file paths canonically and verify that the final path remains inside an explicitly allowed directory. Test URL parsing edge cases rather than blocking one substring in only the query string.
- Run services with limited filesystem permissions so a path-handling flaw cannot read unrelated secrets.

**HTB's lesson:** framework vulnerabilities in the AI stack can cause service crashes or arbitrary file disclosure, and a narrow fix can leave another parsing route open.

**Production perspective:** treat the versions in the lab as historical examples, not as a current vulnerability inventory. Check current vendor advisories and supported versions before making remediation decisions. Verify with version/configuration review and safe tests; do not crash or read files from systems outside explicit assessment scope.

**Notebook decision:** No new Python notebook is needed. The lesson demonstrates vulnerabilities by running deliberately vulnerable framework versions and malformed inputs; an executable notebook could crash local services or access files beyond the intended lab. The study notes preserve the reasoning, version/context, and defensive controls without automating those payloads.

## Section 9 — Introduction to MCP

### What MCP standardizes

**MCP** means **Model Context Protocol** (say the letters “M-C-P”). It standardizes how an AI application connects to external tools and data providers. Before a shared protocol, each integration often needed its own custom connection. MCP provides a common communication pattern, like a standard connector; the service behind that connector still decides what it can do.

The analogy is useful but limited: a standard connector makes integrations easier to build. It does not make every connected service trustworthy or safe. Access, data handling, and tool permissions still need security controls.

### The three architecture roles

- **Host:** the AI application that coordinates the model interaction and manages MCP clients. An editor or desktop assistant can act as a host.
- **Client:** the protocol component created or managed by the host to communicate with an MCP server. In the architecture taught here, each client connects to one server.
- **Server:** a local or remote component that implements MCP capabilities, such as access to files, a database, or an external API. The server is separate from the LLM; it provides the capability, while the host/client coordinate the LLM interaction.

Recall the relationship as: **the host coordinates, the client speaks the protocol, and the server provides capabilities.** Do not assume “server” must mean a separate internet machine; an MCP server may run locally.

### Three server capabilities and who controls them

1. **Prompts** are reusable prompt templates that a user can select and customize with parameters. Example: select a spell-check template and provide a sentence. HTB classifies prompts as **user-controlled**.
2. **Resources** provide information that an application can retrieve as context, such as file contents or database records. The lesson describes them as **read-only** and **application-controlled**. Resources are named by URIs (pronounced “you-are-eyes”); a URI identifies the resource, and an implementation may define its own URI syntax.
3. **Tools** are executable functions the model can request, such as storing a file or making an API request. HTB classifies tool selection as **model-controlled**. The MCP server implements the tool's actual logic; the LLM integration does not itself implement that server-side function.

This control classification is a study aid, not a substitute for an authorization design. A model-requested tool call should still be checked by the host, client, server, and any relevant user-approval policy. Tools can have side effects; resources are presented here as read-only context; prompts guide interaction but do not themselves perform the server action.

### A simplified request-to-answer flow

1. The user sends a request to the host application.
2. The client discovers server tools and resources. Discovery may happen during initialization rather than after the user asks something.
3. The host/client presents available capability descriptions to the LLM in a form it can use.
4. If more context is needed, the client retrieves the appropriate resource and includes it in the model input.
5. The host sends the enriched input to the LLM and receives a generated response.
6. If the response calls for a tool, the client invokes that tool on the MCP server. The result can be added to context and sent back to the LLM. A real application may repeat this step for several tool calls.
7. The host returns the final response to the user.

The security boundary matters: the LLM can propose a tool and arguments, but the host/client performs the protocol call and the server runs the implementation. Review the permissions and validation on the whole path rather than treating the model as the only control point.

### JSON-RPC message types

MCP messages use **JSON-RPC** (say “J-S-O-N remote procedure call,” or “jay-sən R-P-C”), a structured request-and-reply format. JSON-RPC describes the message shape; it does not describe how bytes travel across the connection.

- A **request** starts an operation. It includes an `id` to correlate a reply and a `method` naming the operation; it may include `params` with arguments.
- A **response** uses the same `id` as its request and contains either a `result` or an `error`.
- A **notification** is one-way. It includes a `method` and may include `params`, but has no `id` because the sender does not expect a response.

For example, `tools/call` names an operation to invoke a tool. Its parameters identify the tool and its arguments. The response with the matching ID reports what happened. The ID is correlation bookkeeping; it does not grant permission to perform the operation.

### Transports: how messages travel

A **transport** is the channel used to carry protocol messages between client and server. Keep it distinct from JSON-RPC, which defines the messages.

- **stdio** (say “standard I/O”) uses the operating system's standard input and output streams between local processes. HTB presents it for a client and server running on the same system.
- **Streamable HTTP** uses HTTP requests for client/server communication. The lesson describes HTTP GET and POST, and also **SSE**, “server-sent events,” which can let a server push messages over an HTTP connection without waiting for the client to poll repeatedly.

### MCP lifecycle

1. **Initialization:** the client sends an `initialize` request with its supported protocol version, capabilities, and client information. The server responds with its supported version, capabilities, and server information. The client then sends the `notifications/initialized` notification. They can use compatible versions and capabilities for the session; if the versions cannot be reconciled, the client disconnects.
2. **Operation:** the client and server exchange requests and responses using the capabilities they discovered. Example method names include `prompts/list`, `resources/read`, `tools/list`, and `tools/call`.
3. **Shutdown:** the connection is closed. The section describes no separate MCP shutdown message: stdio streams close for stdio, or the HTTP connection closes for Streamable HTTP.

### COAE exam recall

- **Identify the roles:** host coordinates; client communicates with a server; server implements capabilities.
- **Classify the capability:** a reusable user-selected template is a prompt; read-only context is a resource; an executable operation is a tool.
- **Classify the message:** request starts work and has an ID; response matches that ID and returns a result or error; notification is one-way and has no ID.
- **Separate message format from transport:** JSON-RPC is the format; stdio or Streamable HTTP carries it. SSE is a server-push mechanism described alongside HTTP.
- **Recognize lifecycle order:** initialize and exchange version/capability information, operate using the agreed capabilities, then close the transport.
- **For a security scenario involving a tool:** identify who can request it, which component actually runs it, what authority it has, how its arguments are validated, whether user approval is required, and what external state it can change.

**HTB's lesson:** MCP defines common roles, server capabilities, JSON-RPC messages, transports, and a connection lifecycle for AI applications that use external tools and data.

**Production perspective:** the protocol standardizes communication but does not by itself decide whether a tool is safe, whether an identity is authorized, or whether a response can be trusted. Apply least privilege, validate arguments at the server, limit network and filesystem access, and require approval for sensitive actions where appropriate.

**Notebook decision:** No Python notebook is needed for this theory section. The learning objective is to understand protocol roles and message flow; runnable code would add setup without improving that objective. Keep this material in the shared study notes so later MCP labs can refer back to the same vocabulary.
