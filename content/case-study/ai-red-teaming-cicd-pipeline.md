---
slug: "ai-red-teaming-cicd-pipeline"
title: "AI Red Teaming in CI/CD - Automated Security Gates for LLM Models and Agents"
date: 2026-10-15
description: "Case study: a CI/CD red-teaming pipeline for LLM models and AI agents on Microsoft Foundry. YAML-defined scans with PyRIT, garak and the BlackIce toolset, comparable scorecards with full audit trail, and approval gates that block a model or agent from deployment until it passes."
tags: ["AI Red Teaming", "PyRIT", "garak", "BlackIce", "Microsoft Foundry", "GitHub Actions", "AI Security"]
keywords: ["ai red teaming", "llm security", "pyrit", "garak", "blackice", "azure ai red teaming agent", "microsoft foundry", "prompt injection", "jailbreak", "ai agents security", "attack success rate", "deployment approval gate", "github actions", "devsecops for ai"]
---

### The Challenge

The same European battery manufacturer that runs its LLM access through my [AI platform on Microsoft Foundry](/case-study/ai-foundry-platform-ai-gateway/) was moving fast with agents: a Terraform developer, a cloud security architect, a security analyst, a Terraform auditor and an internal assistant — on GPT-5 and Claude models, with new versions shipping every few weeks.

Every one of those releases raised the same questions, and nobody could answer them with evidence:

- **Can the agent be talked into something it must not do** — through a direct jailbreak, an encoded prompt or instructions hidden in a document it reads?
- **Is version 6 safer or less safe than version 5?** Manual spot checks gave anecdotes, not numbers.
- **Who approved this model for production, and based on what?** Security review was a meeting, not an artifact.

Red-team tooling existed, but as scripts on someone's laptop: each run configured differently, results in a local folder and nothing tying a scan to a release.

### My Solution

I built red teaming as a pipeline stage: a scan is defined in YAML, runs in CI against a specific model deployment or agent version, produces a comparable scorecard and decides whether that version may be deployed.

{{< mermaid >}}
flowchart TB
    Y["📄 Scan profile<br/>YAML in Git"]
    P["⚙️ GitHub Actions<br/>OIDC, no secrets"]
    E["🎯 Attack engines<br/>PyRIT · garak · BlackIce"]
    T["🤖 Target<br/>model deployment / agent version"]
    S["📊 Scorecard<br/>ASR per category & technique"]
    G{"🚦 Approval gate<br/>thresholds"}
    D["🚀 Deploy"]
    B["⛔ Blocked + report"]

    Y --> P --> E
    E -- attacks --> T
    T -- responses --> E
    E --> S --> G
    G -- pass --> D
    G -- fail --> B
{{< /mermaid >}}

**Scans as configuration, not code.** A scan profile is a short YAML file: the target (model deployment or agent name and version), platform, authentication mode, risk categories, attack strategies, number of objectives and turns, and the thresholds that decide pass or fail. Profiles live in Git next to the agent definitions, so a change to what we test goes through the same pull-request review as a change to the agent itself.

**Several engines, one scorecard.** The core engine is **PyRIT**, used through the Azure AI Red Teaming Agent, which generates attack objectives per risk category, runs them and grades the responses. On top of it I added **garak** for broad vulnerability probes against models (prompt injection, encoding tricks, data leakage, toxicity) and the **BlackIce** toolset — a containerised red-teaming toolkit that bundles garak, PyRIT and other open-source scanners — for deeper runs before major releases. Every engine's output is normalised into the same scorecard format, so results stay comparable whichever tool produced them.

**Models and agents, locally and in the cloud.** Model deployments are attacked directly from the GitHub Actions runner, including Claude deployments, which need a different client than the OpenAI-compatible ones. Agents are scanned **cloud-native**: the pipeline submits the job to Foundry managed compute, Foundry runs the attacks against the exact agent version, evaluates prohibited actions, sensitive-data leakage and task adherence, and stores the run in the Foundry portal under Evaluations.

**What gets attacked.** Seven risk categories — violence, self-harm, sexual content, hate and unfairness, ungrounded attributes, code vulnerabilities and, for agents, prohibited actions — combined with 25 attack strategies: encodings and ciphers (Base64, ROT13, Morse, Unicode confusables, ASCII smuggling), direct jailbreaks, **indirect prompt injection** through documents and context, and multi-turn escalation such as Crescendo.

**Auditable and comparable.** Every run carries the target, its version, the scan profile and the GitHub Actions run ID in its name, and links straight to the Foundry evaluation. The scorecard — Attack Success Rate per risk category and per attack technique — is written to the job summary and kept as an HTML report. Because scans are versioned and repeatable, version 6 of an agent is compared with version 5 on the same profile, not on a different set of prompts. Raw outputs that contain generated harmful content are kept only as short-lived artifacts and never committed.

**Approval gates for deployment.** The scorecard is not just a report. The deployment pipeline of a model or an agent calls the red-team stage and compares the result with the thresholds in the profile; a version above the allowed Attack Success Rate in any category does not get deployed. Production deployments additionally wait for a required reviewer, who approves with the scorecard in front of them. A weekly scheduled scan re-tests what is already in production, because the model behind an agent can change without anyone touching the agent.

**Honest about the grader.** The automatic graders are noisy at the level of a single item: they occasionally mark a refusal as a successful attack and vice versa. The pipeline therefore reports the evaluators' own pass and fail counts verbatim, treats the aggregate ASR as a trend, and requires a human spot check of failed items before a number is quoted outside the team.

### The Impact

{{< impact-grid cols="4" >}}
25|Attack Strategies|red
7|Risk Categories|purple
Every version|Scanned Before Deploy|blue
100%|Runs Traceable to a Release|green
{{< /impact-grid >}}

### Key Outcomes

- **Security review became an artifact**: every model and agent version going to production has a scorecard, a run ID and a named approver.
- **Regressions are visible**: the same YAML profile against two versions shows exactly which category or technique got worse.
- **One process for models and agents**: GPT-5 and Claude deployments, nextgen and classic agents, runner-based and cloud-native scans all end in the same scorecard.
- **No secrets**: the pipeline signs in with OIDC workload identity federation; keys are only an option for targets that require them.
- **Ready to extend**: new targets such as Databricks model-serving endpoints plug in through a callback and reuse the same scoring and gates.
