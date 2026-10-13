---
slug: "ai-foundry-platform-ai-gateway"
title: "Enterprise AI platforma na Microsoft Foundry - self-service přístup k LLM přes AI Gateway"
date: 2026-10-01
description: "Případová studie: multi-tenant AI platforma na Microsoft Foundry pro evropského výrobce baterií. Onboarding projektů jako kód, AI Gateway na Azure API Management s povolenými modely a tokenovými kvótami na projekt, náklady po projektech a self-service portál — vše v Terraformu, bez secrets."
tags: ["Microsoft Foundry", "AI Gateway", "APIM", "Terraform", "FinOps", "Platform Engineering"]
keywords: ["microsoft foundry", "azure ai foundry", "ai gateway", "azure api management", "llm platforma", "tokenová kvóta", "alokace nákladů", "terraform", "platform engineering", "self-service onboarding", "entra id", "azure openai", "claude na azure"]
---

### Výzva

Evropský výrobce baterií, součást velké automobilové skupiny, chtěl dát svým týmům přístup k velkým jazykovým modelům na Azure — rodině GPT-5 a Claude přes Microsoft Foundry — aniž by každý tým dostal vlastní klíče ke sdílenému účtu. Požadavky byly ty obvyklé enterprise, jen ostřejší:

- **Izolace**: tým se dostane jen ke svému projektu a jen k modelům, které mu byly schválené.
- **Přiřazení nákladů**: útrata patří týmu, který ji způsobil, i když Azure účtuje nasazení modelu, ne projekt.
- **Self-service**: onboarding nesmí záviset na tom, že platformní inženýr něco naklikal v portálu.
- **Bez secrets**: žádné API klíče a žádné client secrets nikde, ani v pipeline, ani v repozitářích, ani u vývojářů.

Pilot, který platformě předcházel, nic z toho nenabízel: jediný Foundry účet, ručně spravovaný přístup a žádný přehled o útratě po týmech.

### Moje řešení

Platformu jsem navrhl a postavil jako tenkou, plně deklarativní vrstvu nad Microsoft Foundry.

{{< mermaid >}}
flowchart LR
    T["👩‍💻 Tým<br/>Entra ID token"]
    GW["🛡️ AI Gateway<br/>Azure API Management"]
    F["🧠 Microsoft Foundry<br/>GPT-5 · Claude"]
    M["📊 Log Analytics<br/>tokeny po projektech"]
    C["💶 Export nákladů<br/>projekt a den"]

    T -- cesta projektu --> GW
    GW -- managed identity --> F
    GW -. tokenové metriky .-> M
    M --> C
{{< /mermaid >}}

**Onboarding jako kód.** Projekt je pár řádků YAML: název, vlastníci, povolené modely, volitelně limity. Formulář GitHub issue udělá z požadavku automaticky pull request, pipeline ho naplánuje, reviewer mergne a Terraform vytvoří všechno — Foundry projekt, dvě bezpečnostní skupiny v Entra ID (vlastníci a uživatelé), RBAC, Key Vault s klíči projektu ke gateway a produkt na gateway. Odebrání záznamu je offboarding. Globální výchozí hodnoty (limity, role, content safety, tagy) žijí v samostatném souboru, který vyžaduje schválení code ownera, takže přidání projektu zůstává self-service, zatímco změna základní konfigurace ne.

**AI Gateway na Azure API Management.** Každý projekt má před Foundry účtem vlastní produkt v API Managementu. Jeho policy ověří Entra ID token volajícího proti skupinám projektu, vynutí **seznam povolených modelů** (model mimo seznam dostane 403, ať je uvedený v cestě nebo v těle požadavku), uplatní **tokeny za minutu a měsíční tokenovou kvótu**, volitelně prověří prompty přes Azure AI Content Safety a vysílá **tokenové metriky s projektem, modelem a volajícím** jako dimenzemi. Gateway volá Foundry svou managed identitou; lokální autentizace na účtech je vypnutá.

**Identita bez secrets.** Pipeline se přihlašuje přes OIDC workload identity federation a drží nejužší role, které fungují: Role Based Access Control Administrator omezený na Foundry účet a podmíněný třemi definicemi rolí, takže umí přiřadit role projektu a nic jiného. Graph oprávnění jsou omezená na vytváření a vlastnictví skupin a dohledání účtů vlastníků.

**Náklady po projektech.** Azure účtuje po účtech a modelech. Exportní pipeline spojí Cost Management s logy požadavků účtu a tokenovými metrikami gateway a rozdělí denní náklady modelu mezi projekty podle podílu tokenů — výstupní metriky podle completion tokenů, vstupní podle prompt tokenů. Dny, které ještě nejsou vyfakturované, se odhadnou ze stejných jednotkových cen, označí jako odhad a po vyfakturování nahradí. Výsledkem je CSV ve stylu FOCUS po projektech, modelech a dnech, měsíční report a workbook v Azure Monitoru.

**Kill switche.** Dvě named values na projekt v gateway ho zablokují během sekund: jedna pro FinOps při překročení měsíčního budgetu, druhá se přepne automaticky, když odhadovaná denní útrata dosáhne denního limitu projektu. Obě vrací 403 místo 429, protože SDK OpenAI i Anthropic na 429 samy opakují požadavek.

**Self-service portál.** Nad platformou jsem postavil portál pro vlastníky projektů: React frontend a malé API, které čte katalog publikovaný pipeline a ukazuje každému vlastníkovi **jen projekty, jejichž owner skupiny je členem** — tokeny v čase, náklady fakturované i odhadované, stav kvóty a limitů, endpointy, členy týmu a chat s modely projektu. Klíče se čtou a chat běží **jménem přihlášeného uživatele**, takže přístup vynucuje a loguje Key Vault a gateway, ne portál. Role administrátora vidí všechny projekty a rozpad nákladů celé platformy.

### Dopad

{{< impact-grid cols="4" >}}
1 PR|Onboarding až do produkce|blue
0|Secrets v pipeline a repozitářích|green
100%|Spravováno Terraformem|purple
Na projekt|Náklady, kvóta a kill switch|yellow
{{< /impact-grid >}}

### Klíčové výsledky

- **Otestovaná izolace, ne předpokládaná**: token jednoho týmu je na cestě jiného týmu odmítnutý gateway i portálem.
- **Multi-account od prvního dne**: Foundry účty spravované Terraformem i už existující vedle sebe, nasazení GPT-5 a Claude v jednom registru.
- **FinOps může jednat**: každý projekt má řádek nákladů, budgetový přepínač a denní limit a report u každého čísla uvádí zdroj dat.
- **Platformní tým mimo smyčku**: týmy žádají o přístup přes formulář issue; reviewer vidí Terraform plan, ne ticket.
- **Dev až prod se schvalováním**: stejné pipeline plánují na pull request, v dev nasazují po merge a produkci hlídají povinní revieweři.
