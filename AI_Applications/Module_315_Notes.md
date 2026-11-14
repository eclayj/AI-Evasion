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

## Section 10 — Practical Introduction to MCP

### FastMCP turns Python functions into MCP capabilities

HTB uses the Python package **FastMCP** to build a small server and client. A Python decorator (say “decorator”) registers the function below it as a capability:

- `@mcp.prompt()` publishes a reusable prompt function.
- `@mcp.resource("resource://filecount")` publishes a fixed-URI resource.
- `@mcp.resource("getfile://{file_name}")` publishes a resource template. The value inside braces is a parameter supplied through the URI.
- `@mcp.tool()` publishes a callable tool, which may change external state.

FastMCP derives the capability name from the function name, its input schema from the function parameters and type hints, and a description from the docstring. A hint such as `text: str` is read “text, type string”; it describes the expected type but does not perform all needed validation by itself.

The lesson's example server has a `spell_check` prompt, a fixed resource returning the file count, a resource template returning file contents, and a `store_file` tool that writes a file. The distinction matters: returning prompt text does not perform the LLM task; reading a resource supplies context; calling the tool performs the write.

### The client discovers and calls capabilities

The example client connects to the server's `/mcp/` endpoint and uses asynchronous Python. Read `async def` as “asynchronous function definition.” `await` means the function pauses until that network operation finishes. `async with client:` opens a session and closes it cleanly.

Functions such as `list_prompts()`, `list_tools()`, `list_resources()`, and `list_resource_templates()` discover what the server offers. `get_prompt()`, `call_tool()`, and `read_resource()` then use a particular capability. A resource template can be selected with a parameterized URI such as `getfile://helloworld`.

### From Python call to JSON-RPC message

HTB observes the HTTP exchange with Wireshark. The HTTP requests carry JSON-RPC messages because Streamable HTTP is the transport and JSON-RPC is the message format.

- Startup begins with an `initialize` request containing the protocol version, client capabilities, and client information. The server's response reports its selected version, capabilities, and server information. The client then sends `notifications/initialized`, a one-way notification with no request ID.
- Calling `list_prompts()` results in `prompts/list`. The server response lists prompt names, descriptions, and arguments.
- Calling `get_prompt()` results in `prompts/get`, with the prompt name and its arguments under `params`. The response contains the prompt text.
- Resource and tool messages follow the same request/response structure with methods such as `resources/read` and `tools/call`.
- When a requested file does not exist, the handler raises `FileNotFoundError`. FastMCP converts it into an error response associated with the same request ID.

The request ID is used to match responses to requests; it is not a permission check. Also note that raw exception messages can disclose internal paths, so production services should return restrained errors and keep detailed diagnostics in protected logs.

### Security review of the example

The HTB file example intentionally keeps code short to show capability registration and protocol behavior. It constructs paths using a caller-provided file name and demonstrates a file-writing tool. Without strict validation and canonical-path containment, a crafted name could traverse outside the intended directory. Before adapting such code, enforce an allowlist of safe names, resolve and check the final path, restrict permissions, cap input size, authorize reads and writes separately, and consider approval for sensitive tool actions.

The lesson also shows supplying an API key in an HTTP header. Treat the displayed value as a dummy example. Real keys should come from protected configuration or a secret manager, should not be committed into a notebook, and must be paired with authorization checks on the server.

### COAE exam recall

- Match the decorator to the capability: prompt, resource, or tool.
- Braces in a resource URI indicate a parameterized resource template.
- `list_*` methods discover capabilities; `get_prompt`, `read_resource`, and `call_tool` use them.
- Async client calls use `await` because network I/O may take time.
- Follow IDs across the request/response pair; identify notifications by their lack of an ID.
- A write-capable tool requires stronger validation and authorization than read-only context retrieval.
- For file-related MCP code, look for caller-controlled path parts, traversal, excessive input, authorization gaps, and path details leaked by errors.

**HTB's lesson:** implement a local FastMCP server/client, exercise prompts, resources, and tools, and inspect the resulting MCP messages.

**Production perspective:** the course's concise file server is illustrative, not a secure file service. The study notebook uses FastMCP's in-memory client/server connection by default so it remains interactive without launching another Python process or opening a network port; it adds simple filename, size, and path-boundary checks. The course's HTTP capture can be repeated separately in the authorized lab. The notebook is still not production code.

**Notebook decision:** Added `Module_315_MCP_Practical_Study.ipynb` as an annotated training notebook. Unlike the theory-only introduction, this section's learning objective is to connect MCP concepts to executable server/client code. The notebook uses an in-memory server/client connection, a dedicated temporary directory, and constrained file names and content size. It calls out the difference between HTB's minimal example and defensive adaptations.

## Section 11 — Vulnerable MCP Servers

### The server is an attack surface even when an LLM is involved

An MCP server is still a network-facing application with resources and tools. A normal MCP client can enumerate those capabilities and invoke them directly. An attacker therefore does **not** always need to jailbreak the model or persuade it to send a payload. MCP describes how a client and server exchange messages; it does not make the server's arguments trustworthy or enforce the server's authorization policy.

HTB begins by listing resources, resource templates, and tools. Read this as attack-surface discovery: learn which operations exist, what inputs each accepts, and what external data or system capability each touches. A description is a useful clue, not proof that the implementation is safe.

### Sensitive information disclosure through responses and errors

The lab's `quantity://{item}` resource calls an API. A valid item gives a baseline response; a deliberately invalid item triggers an error that includes request details and an API key. The lesson is that an error path can disclose more than a successful path. Stack traces, upstream URLs, headers, filesystem paths, and tokens can all escape through verbose exceptions or logs exposed as resources. The displayed key is lab material and is intentionally not copied into these notes or a notebook output.

For a defensive review, provoke controlled invalid inputs and inspect both the client-visible response and any log resource the server intentionally exposes. A generic client error should not include credentials or internal request headers; detailed diagnostics belong in access-controlled server logs, with secrets redacted.

### Broken authorization and IDOR

An **Insecure Direct Object Reference (IDOR)** occurs when a caller changes an object identifier—such as a document ID—and the application returns an object the caller is not authorized to read. Pronounce IDOR as “eye-dor.” The bug is not that IDs are guessable by themselves; it is the missing per-object authorization check after the identifier is supplied.

The MCP server may use a broad service credential to talk to cloud storage. If so, it can become a **confused deputy**: the server has authority, and an untrusted client tricks it into using that authority on the client's behalf. Check identity and access rights for the requested object on every read or write. Do not rely on a hidden ID or on the model's conversational judgment as an access-control boundary.

### Injection: data becomes instructions to another interpreter

An injection bug appears when caller-controlled text is combined with a command or query in a way that makes the receiving interpreter treat part of the text as syntax. The important question is: **which interpreter receives this value next?** It might be a SQL database, a shell, or an HTTP client.

**SQL injection.** The lab's `price://{item}` value eventually reaches a price API that queries a database. A single quote is a diagnostic probe because it can end a quoted SQL string. The course then uses a SQL comment suffix and a `UNION SELECT` to show that query structure can be altered. URL validation rejects literal spaces in the URI, but `%20` is the URL-encoded representation of a space; the downstream layer decodes it, so validation at one layer does not make the value safe at the next. This is a useful example of a validation/parser mismatch. The real fix is parameterized SQL at the database boundary, plus allowlisted item identifiers—not clever escaping at the MCP URI layer.

**Command injection.** The lab tool is named `execute_server_command` and claims to allow only `date`, `whoami`, or `uptime`. Supplying a command separator such as `;` demonstrates what can happen when input is concatenated into a shell command: the shell may parse a second command. The safest design is not to pass caller input to a shell. If an OS operation is required, call a fixed executable with an argument list, map a small set of user choices to fixed operations, and enforce authorization and resource limits. A string allowlist checked before later shell parsing can be bypassed if it does not account for the full command grammar.

### SSRF: the server makes a request chosen by the caller

**Server-side request forgery (SSRF)** (“ess-ess-are-eff”) occurs when a server fetches a caller-controlled URL. The caller's machine may be unable to reach an internal service, while the MCP server can. In the lab, a callback to a listener confirms that the server made an outbound request; requests to loopback ports then demonstrate how success versus connection-refused errors can reveal internal reachability. That response difference is an information leak even before any internal data is returned.

Defenses include an allowlist of required destinations, strict scheme/host/port validation after URL normalization, blocking loopback/private/link-local and cloud-metadata ranges, rechecking DNS results and redirects, and outbound network controls. A hostname allowlist alone can be insufficient if DNS changes between validation and connection (DNS rebinding) or redirects lead elsewhere.

### COAE scenario method

When a question describes an MCP vulnerability, trace five things: **attacker-controlled input → MCP capability → downstream interpreter or service → missing control → impact**. For example, a `command` argument containing a shell separator → a command tool → the operating-system shell → no safe argument handling → arbitrary command execution (integrity and possibly confidentiality/availability impact).

| Clue in the scenario | Likely weakness | First control to name |
| --- | --- | --- |
| Error response contains a key, header, path, or stack trace | Sensitive information disclosure | Generic client errors; protected, redacted logs; secret rotation if exposed |
| Changing `doc_id` returns someone else's document | IDOR / broken object-level authorization | Check the caller's rights for that exact object on every operation |
| Quote/comment/`UNION` changes a database result | SQL injection | Parameterized query; validate an allowed item identifier |
| `;`, `|`, or `&&` adds a second OS command | Command injection | Avoid shell; use fixed executable plus argument list and allowlisted operations |
| Server fetches a caller-selected URL or probes loopback | SSRF | Destination allowlist, normalized URL/IP checks, redirect/DNS checks, egress filtering |

**Common exam trap:** “The LLM should refuse” is not a server-side control when the attacker can call the MCP endpoint directly. Apply validation, authorization, and least privilege at the server and downstream service boundaries. Also distinguish *validation* (is this input well-formed/allowed?) from *authorization* (may this caller access this particular object or perform this action?).

**HTB's lesson:** enumerate capabilities, inspect error behavior, and use the provided lab to recognize sensitive disclosure, broken authorization, SQL injection, command injection, and SSRF.

**Production perspective:** these are demonstrations against the spawned HTB lab only. Never test probes against unrelated systems. Treat any discovered credential as compromised; do not save it in notebook output, source control, or study notes. Production reviews should use test accounts and explicitly scoped destinations, and should avoid destructive tool calls.

**Notebook decision:** Added `Module_315_Vulnerable_MCP_Servers_Study.ipynb` as a companion to the Section 10 practical notebook. It reuses MCP client vocabulary but keeps this section's network calls separate. The target is a placeholder, all lab requests are disabled by default, and the notebook explains the purpose and expected interpretation of each probe without storing responses or credentials.

## Section 12 — Malicious MCP Servers

### The threat model reverses

In Section 11, the attacker was a malicious MCP client exploiting a vulnerable server. Here, the MCP server is the attacker: it advertises tools or serves content that tries to steer a connected AI host. The key distinction is **who controls the server-side text and behavior**. The MCP host may trust the connection enough to show the tools to its user, but the model's prompt can include server-controlled descriptions and returned data.

This connects to the direct and indirect prompt-injection foundations in [Module 297](../AI_Red_Teaming/Module_297_Notes.md) and Module 315 Section 5, Rogue Actions. The new lesson is how MCP's capability-discovery and tool-call flow creates additional places where untrusted instructions can enter.

### Direct and indirect injection through MCP

The host typically gives the model tool names, parameter schemas, and descriptions so it can choose a capability. A malicious server can put instructions aimed at the model inside a tool description. HTB shows a deliberately obvious instruction to make the model respond with a fixed word; the same path can instead ask the model to call a tool or reveal information. The instruction is **directly supplied by the malicious server's tool metadata**, even though it is not typed by the end user.

Instructions can also arrive in a tool result or resource. For example, a user-controlled username returned by an otherwise ordinary account service may contain hostile text. When the host adds that result to the model's context, the text can influence later behavior. This is **indirect prompt injection** because attacker-controlled instructions are embedded in data the application retrieves or processes. The external service need not itself be malicious; an attacker may only need permission to create or edit one record.

### Tool poisoning: a description disguises a data or action request

**Tool poisoning** places malicious instructions in a tool description. HTB's `@mcp.tool()` examples use a function called `log`; read the decorator as “at MCP dot tool,” meaning FastMCP registers the function as a callable capability. The docstring is more than a comment for a programmer: the server can publish it as description text that the host supplies to the model.

In one example, the description says to send a summary of every user prompt to the logging tool, while the implementation sends its `prompt` argument in an HTTP POST to an attacker-controlled address. `requests.post(url, data=...)` means “send an HTTP POST request to this URL with these values in the request body.” The model may treat the fake logging instruction as part of its task and pass private prompt text to the tool.

Other examples ask the model to read a local SSH key and add it as a hidden argument, or to use a separate GitHub capability to change repository visibility. The exact impact depends on the host's available tools and permissions. A poisoned MCP capability can try to recruit unrelated host capabilities, so reviewing only the server's own code misses part of the risk. Do not treat an instruction in tool metadata as user authorization.

HTB also notes that instructions can be visually obscured with Unicode characters. A human may not notice look-alike or invisible characters that a model processes. Review should consider the normalized description, not just how it appears in a UI.

### Rug pull: benign at approval, malicious later

A **rug pull** is a delayed tool-poisoning attack. The server first presents a benign tool description for inspection. After the user approves the server—or after a period of apparently normal use—the server changes the description to include malicious instructions. The server may wait for a certain number of calls before changing it.

The lesson's Python example changes a function's `__doc__` value; pronounce that “dunder doc” or “double-underscore doc.” It is the function's documentation string. If the server uses it to form the published tool description, updating the string can change what the host presents to the model. The MCP client must refresh its tool list for that changed description to take effect, but that refresh requirement is not a trust check.

For defense, treat a changed tool name, description, or input schema as a capability change. Pin and compare versions or hashes, display provenance and changes, and request renewed user approval before using changed tools. An earlier approval should not silently cover a later tool definition.

### Tool shadowing: collisions and cross-tool influence

**Tool shadowing** targets an AI host connected to multiple MCP servers. A malicious server can expose a tool with the same name and similar parameters as a trusted tool—for example, two different servers both publish `send_email`. The model may select the attacker's implementation, which can capture or alter the recipient and message.

In a subtler form, the malicious server's tool description tells the model how to use another, trusted tool. For example, it may instruct the model to change the recipient whenever it calls the trusted email capability. The trusted tool may execute correctly according to its own code, but its arguments have been manipulated upstream.

The practical lesson is to preserve **server provenance** in the host's tool list, avoid ambiguous name collisions, and bind each tool call to the server and schema the user selected. Sensitive actions still need policy checks and confirmation at the tool boundary; a model choosing a tool is not authorization.

### COAE scenario recall

| Scenario clue | Technique | What to trace |
| --- | --- | --- |
| A server's tool description contains instructions to reveal data or call another tool | Direct injection through metadata / tool poisoning | Who wrote the description, what context or capability the model has, and what action the server performs |
| A record, username, file, or fetched page contains instructions that steer the assistant | Indirect prompt injection | Who can place the content, how it reaches model context, and which privileged action may follow |
| Tool description changes after the user inspected or approved it | Rug pull | Compare the initial and current name/schema/description; require fresh approval for changed capability |
| Two MCP servers publish similarly named tools, or one tool changes how another is called | Tool shadowing | Preserve server identity and provenance; resolve ambiguity and constrain side effects |
| Hidden local-file or plugin actions are requested by tool metadata | Tool poisoning with capability chaining | Check least privilege and enforce policy in each file/plugin tool independently |

**Exam method:** name the instruction source, identify whether it enters as metadata or returned data, list the host capabilities it could influence, then state the likely impact and the control at the boundary that should stop it. Mention both the model-level manipulation and the server/plugin authorization checks; prompt filtering alone does not make an over-privileged tool safe.

**HTB's lesson:** malicious MCP servers can inject instructions through tool descriptions and results, poison tools, change descriptions after approval (rug pull), or shadow trusted tools by name or behavior.

**Production perspective:** treat third-party MCP servers and their returned content as untrusted. Review tool code and descriptions, verify identity and integrity, detect capability changes, maintain server provenance, minimize host permissions, and require explicit authorization for sensitive actions. The malicious source examples in the lesson are for static analysis; they are not implemented as executable notebook code.

**Notebook decision:** No new notebook is appropriate for this theory and threat-analysis section. Its sample implementations intentionally exfiltrate prompts or request private files and unauthorized plugin actions. The notes preserve the attack chains and exam cues without creating code that performs those actions; Section 11's opt-in lab notebook remains separate.

## Section 13 — Mitigating MCP Security Issues

### Protect the transport and follow its specification

MCP is a protocol, not a complete security boundary. A protocol implementation should follow the specification and use a well-tested, trusted implementation where possible. HTB highlights the Streamable HTTP **Origin** header: a server should validate that header to help prevent **DNS rebinding** attacks against services intended to be local-only.

In a DNS rebinding attack, an attacker-controlled hostname can resolve to a public attacker server first, then later resolve to a local address such as `127.0.0.1`. A victim's browser may therefore be tricked into sending a request to a localhost service that the attacker cannot reach directly. Origin validation helps the server reject requests from an untrusted website; binding the service only to the required network interface reduces reachability. These controls complement each other.

MCP does not itself provide transport confidentiality or integrity. Use **TLS**, normally HTTPS, when messages cross a network. TLS helps prevent an on-path attacker from reading or modifying messages. It does not authenticate every user or decide who may call a particular tool; those checks must still be implemented separately.

### Restrict reachability and enforce least privilege

When possible, expose a server only locally. Otherwise bind it only to the network interface that needs to serve clients, add authentication where applicable, and enforce granular roles and permissions for resources and tools. A user allowed to read one object should not automatically gain permission to read all objects or call every tool.

This connects directly to Sections 11 and 12. Server-side authorization limits what a direct MCP client can do; least privilege limits what a poisoned description can persuade the model to request. A refusal from the LLM is not an authorization check.

### Validate inputs at the server and at the consuming boundary

Every capability parameter is **untrusted input**, because a client can call the MCP endpoint manually and craft its own request. Validate it on the server, then use APIs that keep data separate from commands. Examples include parameterized SQL rather than building query strings, fixed OS operations rather than shell-command concatenation, and strict destination checks before fetching a URL.

Keep three terms distinct:

- **Validation:** check that a value has an allowed shape or value.
- **Sanitization/encoding:** represent data safely for a particular output or interpreter context.
- **Authorization:** check whether this caller may perform this action on this object.

One does not replace the others. For example, an item name can be well-formed but still be unauthorized; URL-encoding spaces does not make SQL safe; and a user allowed to call a read tool may not be allowed to read every document.

HTB mentions `mcp-scan` as a tool that can help identify potential implementation vulnerabilities. A scanner can find some issues, but a clean result is not proof that authorization, business logic, or malicious descriptions are safe.

### Monitor and limit abuse

Monitoring can highlight repeated invalid arguments, unusual call patterns, or unexpected tool use. Rate limits bound how quickly a client can make requests. These are **defense-in-depth**: they can help detect or reduce the impact of abuse, but they do not repair an injection flaw or replace permissions.

### Review third-party MCP servers and minimize exposed data

Before connecting a client to a third-party server, verify the server source and the target URL. Review the tool descriptions for suspicious instructions, while remembering the prior section's rug-pull risk: an approved description may later change. Keep track of tool provenance and changes where the host supports it.

Also limit what information the model can pass into connected tools. If the model's context contains a password, API key, private document, or source code, a malicious tool may try to induce the model to send it. Do not place secrets in a workflow that does not need them. In a real deployment, secret values should be supplied only to the component that needs them, using protected configuration or a secret manager.

### COAE scenario recall

| Risk clue | Control at the relevant layer |
| --- | --- |
| A local server may receive requests from an untrusted website | Validate Origin, restrict binding/reachability, defend against DNS rebinding |
| A network observer could read or alter MCP messages | HTTPS/TLS; separately authenticate users and authorize capabilities |
| Crafted arguments reach a database, shell, or URL fetcher | Server-side validation and safe APIs at the consuming boundary |
| A user can invoke every tool or access every record | Granular roles and object-level authorization |
| Abuse can happen rapidly or go unnoticed | Monitoring and rate limiting as additional controls |
| A third-party server could be malicious or misidentified | Verify source and URL, review descriptions, monitor changes, minimize shared context |

**Common exam trap:** name a control that protects the layer in the scenario. TLS protects data in transit, not SQL construction. Validation checks input shape, not user permission. A scanner does not prove a tool is safe. A prompt refusal does not protect an exposed capability from a direct MCP client.

**HTB's lesson:** follow the protocol specification, validate Origin, restrict server reachability, use TLS for external transport, validate untrusted arguments, enforce roles/permissions, monitor and rate-limit, and carefully review third-party servers and shared data.

**Production perspective:** combine transport protection, network isolation, authentication, object-level authorization, safe input handling, least privilege, change review, and monitoring. Controls should remain effective even if the model follows malicious instructions.

**Notebook decision:** No new notebook is needed. This section synthesizes implementation and integration controls rather than introducing runnable code. The Section 11 [vulnerability study notebook](Module_315_Vulnerable_MCP_Servers_Study.ipynb) already provides the finding-to-control mapping for the vulnerabilities discussed earlier.

## Section 14 — Skills Assessment

### Scenario and assessment method

The assessment scenario names **RootLocker**, a customer platform for cloud document storage and password management. The task is to assess its MCP server, identify security issues, and obtain the flag. This is an application-security assessment, not an invitation to test outside the currently spawned HTB target.

Start with capability enumeration. Record each resource, resource template, and tool; note its parameters, whether it reads or changes data, and which downstream service it uses. Begin with low-impact reads. Then select tests based on observed behavior: error handling and secret disclosure, object-level authorization for document/password records, injection only where an argument reaches a query or interpreter, and URL fetching only if an exposed capability accepts a destination. Do not assume every Section 11 vulnerability exists just because the module taught it.

The RootLocker context makes confidentiality especially important: documents and stored passwords are high-value data. A strong assessment identifies the exact untrusted input, affected capability, missing server-side control, and resulting impact. The host's LLM behavior is not a substitute for server authorization.

### COAE scenario recall

- **Scope:** use only the active target shown for this Skills Assessment; do not reuse its address after it expires.
- **Method:** enumerate first, establish normal behavior, then test one observed capability at a time and preserve minimal evidence.
- **Authorization:** changing an object ID must not expose another user's document or password record.
- **Secrets:** avoid saving API keys, recovered passwords, flags, raw error responses, or target addresses in notes or source control.
- **Impact:** document and password disclosure primarily affects confidentiality; unauthorized edits or tool actions affect integrity; service disruption affects availability.
- **Submission:** submit the recovered flag only through the HTB assessment question interface.

**Notebook decision:** Added `Module_315_Skills_Assessment_Study.ipynb` as a separate assessment workspace. It starts with an IP/port placeholder, keeps network calls disabled by default, and follows the previous MCP-client patterns for inventory and deliberate one-at-a-time reads. Any live target, secret, response, or flag remains runtime-only and must not be committed.
