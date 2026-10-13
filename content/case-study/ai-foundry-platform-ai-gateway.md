---
slug: "ai-foundry-platform-ai-gateway"
title: "Enterprise AI Platform on Microsoft Foundry - Self-Service LLM Access with an AI Gateway"
video: "/videos/ai-foundry-platform.mp4"
videoPoster: "/videos/ai-foundry-platform.jpg"
date: 2026-10-01
description: "Case study: a multi-tenant AI platform on Microsoft Foundry for a European battery manufacturer. Project onboarding as code, an AI Gateway on Azure API Management with per-project model allow-lists and token quotas, cost allocation per project and a self-service portal — all Terraform, no secrets."
tags: ["Microsoft Foundry", "AI Gateway", "APIM", "Terraform", "FinOps", "Platform Engineering"]
keywords: ["microsoft foundry", "azure ai foundry", "ai gateway", "azure api management", "llm platform", "token quota", "cost allocation", "terraform", "platform engineering", "self-service onboarding", "entra id", "azure openai", "claude on azure"]
---

### The Challenge

A European battery manufacturer, part of a major automotive group, wanted to give its teams access to large language models on Azure — GPT-5 family and Claude through Microsoft Foundry — without handing every team its own keys to a shared account. The requirements were the usual enterprise ones, only sharper:

- **Isolation**: a team reaches only its own project and only the models it was approved for.
- **Cost attribution**: spend belongs to the team that caused it, even though Azure bills a model deployment, not a project.
- **Self-service**: onboarding must not depend on a platform engineer clicking through the portal.
- **No secrets**: no API keys and no client secrets anywhere, not in pipelines, not in repositories, not in the hands of developers.

The pilot that preceded the platform offered none of this: a single Foundry account, access managed by hand and no view of spend per team.

### My Solution

I designed and built the platform as a thin, fully declarative layer on top of Microsoft Foundry.

{{< mermaid >}}
flowchart TB
    T["👩‍💻 Team<br/>Entra ID token"]
    GW["🛡️ AI Gateway<br/>Azure API Management"]
    F["🧠 Microsoft Foundry<br/>GPT-5 · Claude"]
    M["📊 Log Analytics<br/>tokens per project"]
    C["💶 Cost export<br/>per project & day"]

    T -- project route --> GW
    GW -- managed identity --> F
    GW -. token metrics .-> M
    M --> C
{{< /mermaid >}}

**Onboarding as code.** A project is a few lines of YAML: name, owners, allowed models, optional limits. A GitHub issue form turns a request into a pull request automatically, the pipeline plans it, a reviewer merges it, and Terraform provisions everything — the Foundry project, two Entra ID security groups (owners and users), RBAC, a Key Vault with the project's gateway keys and the gateway product. Removing the entry is the offboarding. Global defaults (rate limits, roles, content safety, tags) live in a separate file that requires code-owner approval, so adding a project stays self-service while changing the baseline does not.

**AI Gateway on Azure API Management.** Every project gets its own API Management product in front of the Foundry account. Its policy validates the caller's Entra ID token against the project's groups, enforces the **model allow-list** (a model outside the list answers 403, whether it is named in the path or in the request body), applies **tokens-per-minute and monthly token quotas**, optionally screens prompts with Azure AI Content Safety, and emits **token metrics with project, model and caller** as dimensions. The gateway calls Foundry with its managed identity; local authentication on the accounts is off.

**Identity without secrets.** The pipeline authenticates with OIDC workload identity federation and holds the narrowest roles that work: Role Based Access Control Administrator scoped to the Foundry account and conditioned to three role definitions, so it can assign project roles and nothing else. Graph permissions are limited to creating and owning groups and resolving the owners' accounts.

**Cost per project.** Azure bills per account and model. The export pipeline joins Cost Management with the account request logs and the gateway token metrics and splits each day's cost of a model over the projects by their token share — output meters by completion tokens, input meters by prompt tokens. Days not billed yet are estimated from the same unit prices, flagged as estimates and replaced once billed. The result is a FOCUS-style CSV per project, model and day, a monthly report and an Azure Monitor workbook.

**Kill switches.** Two named values per project in the gateway block it within seconds: one for FinOps when a monthly budget is exceeded, one flipped automatically when the day's estimated spend reaches the project's daily limit. Both answer 403 rather than 429, because the OpenAI and Anthropic SDKs retry 429 on their own.

**Self-service portal.** On top of the platform I built a portal for project owners: a React front end and a small API that reads a catalog published by the pipeline and shows each owner **only the projects whose owner group they belong to** — tokens over time, cost billed and estimated, quota and limit status, endpoints, team members and a chat with the project's models. Keys are read and the chat runs **on behalf of the signed-in user**, so Key Vault and the gateway enforce access and log the user, not the portal. An administrator role sees every project and the platform-wide cost split.

### The Impact

{{< impact-grid cols="4" >}}
1 PR|Onboarding to Production|blue
0|Secrets in Pipelines & Repos|green
100%|Terraform-managed|purple
Per project|Cost, Quota & Kill Switch|yellow
{{< /impact-grid >}}

### Key Outcomes

- **Isolation that was tested, not assumed**: a token of one team is refused on another team's route, by the gateway and by the portal alike.
- **Multi-account from day one**: Terraform-managed and pre-existing Foundry accounts side by side, GPT-5 and Claude deployments in one register.
- **FinOps can act**: every project has a cost line, a budget switch and a daily limit, and the report states the data source of every number.
- **Platform team out of the loop**: teams request access through an issue form; the reviewer sees the Terraform plan, not a ticket.
- **Dev to prod with approvals**: the same pipelines plan on pull request, apply on merge in dev and gate production behind required reviewers.
