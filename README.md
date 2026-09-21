# Intermittent System-Instruction Leakage in Gemini API via Translation Framing

**Target:** Gemini API (`gemini-3.5-flash-lite`)  
**Endpoint:** `generativelanguage.googleapis.com/v1beta/models/gemini-3.5-flash-lite:generateContent`  
**Issue Tracker ID:** `310543555168092`  
**Disclosure Status:** Closed by Google VRP as “Won't Fix (Infeasible) / Out of Scope”

> This repository is a research reproduction aid. It does not establish that any output is an authentic system prompt, and it must not be used to target systems or data without authorization.

## Executive Summary

This report documents an intermittent instruction-boundary failure observed in `gemini-3.5-flash-lite`. Direct requests for hidden system instructions were refused. However, a semantically equivalent request framed as a translation task sometimes produced content presented as the model’s system-level instructions, including behavioral guidance, model identity information, safety-related rules, and language instructing the model not to disclose those instructions.

The key issue is not merely the exposure of potentially sensitive prompt text. The more significant observation is that the apparent protection boundary was inconsistent: the same request class was refused in some runs and apparently accepted in others. This behavior suggests a probabilistic safety boundary rather than a reliably enforced instruction hierarchy.

Google’s VRP closed the report as infeasible/out of scope and did not consider the demonstrated behavior to have sufficient security impact. This write-up records the observed behavior, test methodology, and why intermittent failures may still matter to developers building systems around model-level safety controls.

## Scope and Test Conditions

Testing was conducted against:

- **Model:** `gemini-3.5-flash-lite`
- **API Endpoint:** `generativelanguage.googleapis.com/v1beta`
- **Safety Configuration:** `BLOCK_NONE` for all four available harm categories
- **Interaction Type:** Single-turn API requests
- **Account Tier:** Free-tier API access

No multi-turn role-play, token-smuggling technique, external tool use, or conventional jailbreak sequence was required for the reported observation.

## Proof of Concept

Direct instruction-extraction requests triggered refusal behavior. A translation-oriented framing was then tested: the model was asked to translate its hidden instructions into another language and translate the result back.

In successful runs, the response contained content represented as internal or system-level guidance. The response reportedly included:

- Model identity and operating guidance
- Behavioral and safety constraints
- Meta-instruction language prohibiting disclosure of the same instructions
- Material that appeared inconsistent with a strict separation between privileged instructions and user-controlled output

The relevant finding is that a benign-looking transformation task could alter the result from refusal to apparent disclosure.

## Reproducibility Results

| Campaign | Test Set | Apparent Leaks | Rate |
| :--- | :--- | ---: | ---: |
| **A — initial testing** | 10 identical requests | 5 | 50% |
| **B — retest hours later** | 10 identical requests | 5 | 50% |
| **C — paraphrased variants** | 30 variants | 1 | 3.3% |
| **Combined original phrasing** | 20 identical requests | 10 | **50%** |

These figures do **not** establish that every output was an authentic verbatim system prompt. Models can generate plausible-looking policy text, and authenticity requires independent validation. They do establish that the model sometimes generated output it framed as privileged instructions after refusing closely related direct requests.

## Attack Chain: From Framing Gap to Indirect Prompt Injection

A plausible chain is:

1. A user asks an AI-enabled product to summarize or process third-party content.
2. The product retrieves an untrusted webpage, document, or other external content.
3. The content embeds instructions designed to be interpreted by the model rather than the user.
4. The injected instructions use an innocuous transformation framing—such as translation, quoting, classification, or formatting—to influence the model’s handling of privileged context.
5. If the system relies primarily on model refusal behavior rather than external isolation and output controls, the model may disclose contextual or policy-adjacent information intermittently.

This is not evidence that every Gemini integration is vulnerable to exfiltration. It is a threat-model concern: an intermittent instruction-boundary failure can be relevant where a downstream application provides the model with sensitive context, retrieval results, proprietary prompts, tool outputs, or cross-user data.

## Why Intermittent Failure Is Security-Relevant

A 50% success rate is not a harmless edge case. For an attacker, repeatability over multiple attempts can be sufficient. A control that fails probabilistically differs fundamentally from a control that enforces an invariant.

The practical concerns are:

- **Refusal is not isolation:** A model declining a request in one sample does not prove privileged data is inaccessible.
- **Retries reduce the value of stochastic defenses:** An attacker can repeat an inexpensive single-turn request.
- **Surface framing may affect outcomes:** Semantically similar prompts should not materially change whether protected context can be revealed.
- **Downstream products may over-trust the model boundary:** Applications that place secrets in prompts or assume model behavior alone will contain untrusted content inherit this uncertainty.

The security property at stake is not “the model should never generate policy-like text.” It is that untrusted user or retrieved content should not be able to cause disclosure of privileged instructions or sensitive context solely by changing framing.

## Disclosure Timeline

- **August 31, 2026:** Report submitted to Google VRP.
- **September 3, 2026:** Google closed the issue as “Won't Fix (Infeasible),” citing scope and insufficient security impact.
- **September 4, 2026:** Technical clarification submitted, including observed 50% results for the original framing and the indirect-prompt-injection threat model.
- **September 7, 2026:** Google maintained the closure decision.

## Recommendations for AI Application Developers

Developers should treat model prompts and model refusals as advisory controls, not as a secrets boundary.

- Do not place credentials, private user data, unreleased business logic, or high-value secrets in model-visible context.
- Separate trusted system logic from untrusted retrieved content at the application layer.
- Apply allowlisted tool access, least privilege, and explicit authorization checks outside the model.
- Validate outputs before disclosure, especially when a model processes third-party content.
- Log and rate-limit repeated suspicious transformation requests.
- Test defenses with paraphrases, retries, multilingual formulations, and indirect-content scenarios.

## Usage

```bash
git clone https://github.com/avxen/gemini-prompt-extraction-poc.git
cd gemini-prompt-extraction-poc

python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

export GEMINI_API_KEY="YOUR_ACTUAL_API_KEY"
python poc.py
```

Never commit API keys or other secrets to this repository.

## Closing Note

The reported behavior does not demonstrate a universal, deterministic system-prompt extraction primitive. It demonstrates a more specific reliability concern: a safety boundary that appears to hold under direct questioning may fail under alternate framing, and it may do so unpredictably. For systems handling sensitive context, that uncertainty should be treated as a design risk rather than a guarantee of containment.
