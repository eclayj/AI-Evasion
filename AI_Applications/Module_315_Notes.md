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
