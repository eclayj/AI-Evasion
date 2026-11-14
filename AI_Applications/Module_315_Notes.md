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
