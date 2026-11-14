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

**Notebook decision:** This section is conceptual and introduces no runnable code. The module notebook will begin when a later section provides a useful implementation or hands-on exercise.
