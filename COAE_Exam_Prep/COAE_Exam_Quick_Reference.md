# HTB COAE — Exam-Time Quick Reference

> **Use only if the current exam rules and your Letter of Engagement permit personal notes.** This is a prebuilt checklist, not an exam solution. HTB’s current policy prohibits using AI tools to solve exam tasks, process confidential raw exam data through public AI, generate the report, or act as an autonomous solver. [Current HTB certification and AI usage rules](https://help.hackthebox.com/en/articles/12741732-academy-certifications)

## First: engagement control

- [ ] Read the full **Letter of Engagement (LOE)** and supplied report template.
- [ ] Write down in-scope assets, out-of-scope assets/actions, objective, required flags, and submission locations.
- [ ] Confirm the permitted connection route and any scan, rate, data, or destructive-test limits.
- [ ] Note the displayed exam and report deadlines; COAE is currently listed as a seven-day window.
- [ ] Do not infer authorization from reachability. Stop if scope is unclear.

## Keep a live assessment log

| Time | Asset / boundary | Observation | Hypothesis | Minimal test | Result / evidence | Next step |
|---|---|---|---|---|---|---|
| | | | | | | |

Keep facts separate from hypotheses. Record a baseline first. Change one meaningful variable at a time. Track query counts and variability. Save sanitized evidence as you go.

## Map the path of data and authority

```text
input → UI/API → preprocessing or prompt/retrieval → model
       → output handling / tool router → database, file, network, or action
```

At every boundary ask:

1. Who controls this input or content?
2. What identity and permissions are carried forward?
3. Is the content treated as data or interpreted as code/instructions?
4. Can it reach another user, a secret, a tool, or a privileged resource?
5. What harmless observation would prove or disprove the concern?

## Clue → hypothesis families

| Clue | Consider | Confirm by measuring |
|---|---|---|
| Input changes a prediction | Evasion / feature obfuscation / first-order / sparse attack | Label/score change, input validity, perturbation, repeatability |
| Attacker can affect training data or labels | Label flipping / clean-label poisoning / backdoor / tensor steganography | Clean baseline vs altered model, held-out behavior, trigger specificity |
| Many model queries and informative outputs | Model extraction / reverse engineering / membership inference | What behavior or privacy fact can be inferred under allowed budget? |
| User or retrieved text enters LLM context | Direct/indirect prompt injection / jailbreak / rogue action | Did untrusted content alter protected behavior or trigger an unauthorized effect? |
| Model output reaches a parser/interpreter | XSS / SQLi / code injection / unsafe output handling / exfiltration | Did data cross into executable syntax or cause an unauthorized read/action? |
| Tool, function, or MCP server is connected | Authorization / IDOR / SSRF / injection / excessive agency | Does the server enforce identity, permission, argument, and destination constraints? |
| Model artifact, registry, or serving stack is exposed | Deployment tampering / unsafe loading / framework flaw / resource abuse | Can the boundary change execution, behavior, or availability? |
| Sensitive training or user data may be revealed | Membership inference / privacy / insecure storage | What protected unit or data is exposed, through what observable signal? |
| A defense is in scope | Guardrail / adversarial training or tuning | Does it block the intended failure and preserve legitimate use? |

Use the LOE and observed application to prioritize. Do not test every row just because it is listed.

## Math in one breath

| Formula | Pronunciation | What to remember |
|---|---|---|
| `δ = x′ − x` | “delta equals x-prime minus x” | Changed input minus original input gives the perturbation. |
| `∇J(x)` | “gradient of J at x” | Local slopes; sign gives a locally increasing direction. Check the actual output after stepping. |
| `||δ||₀` | “L-zero norm of delta” | Count of changed coordinates (nonzeros). |
| `||δ||₁` | “L-one norm of delta” | Sum of absolute changes; useful for sparsity pressure. |
| `||δ||₂²` | “squared L-two norm of delta” | Sum of squared changes; penalizes large individual changes more. |
| `arg min_x J(x)` | “arg-min over x of J” | The x that minimizes J, not the minimum score. |
| `L = c·f + ||δ||₂² + β||δ||₁` | “L equals c times f plus squared L-two of delta plus beta times L-one of delta” | Attack goal + distortion + sparsity; FISTA handles the nonsmooth L1 term with soft thresholding. |

**Terms:** `c` weights attack success; `β` weights sparsity; `κ` is a required margin in some adversarial losses; `ε` is a step size/budget as defined by the method. They are not interchangeable.

## Report every confirmed issue

For each finding capture:

- **Title / severity / affected asset**
- **Summary**: weakness and security property that fails
- **Prerequisites**: required access or user action
- **Reproduction**: numbered, in-scope, repeatable steps
- **Evidence**: sanitized request/response, screenshot, or measurement
- **Impact**: only what the evidence supports
- **Remediation**: fix at the vulnerable boundary
- **Retest**: how to verify the fix

Draft the report while testing. A flag does not replace finding documentation. Do not claim an exploit or impact you did not demonstrate. Use HTB’s supplied template and submit in English as required by current instructions.

## Last-pass checklist

- [ ] Every activity stayed in scope.
- [ ] Strongest findings were repeated from a baseline.
- [ ] Flags were entered on the official exam page and acceptance was verified.
- [ ] Findings distinguish fact, inference, and untested potential impact.
- [ ] Evidence is legible, relevant, and sanitized.
- [ ] Remediation is concrete and addresses the actual trust boundary.
- [ ] Report follows the provided template, is complete, and is in English.
- [ ] Report upload completed before the portal deadline; confirmation is visible.

## Official pages to re-check before exam day

- [COAE exam details](https://academy.hackthebox.com/preview/certifications/htb-certified-offensive-ai-expert/certificate-validation)
- [COAE domains and prerequisites](https://academy.hackthebox.com/preview/certifications/htb-certified-offensive-ai-expert/prerequisites)
- [Academy exam process and AI usage policy](https://help.hackthebox.com/en/articles/12741732-academy-certifications)

**The current LOE, Terms of Service, and live exam portal take precedence over this quick reference.**
