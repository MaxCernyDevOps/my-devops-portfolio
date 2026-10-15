---
slug: "ai-red-teaming-cicd-pipeline"
title: "AI red teaming v CI/CD - automatické bezpečnostní brány pro LLM modely a agenty"
date: 2026-10-15
description: "Případová studie: CI/CD pipeline pro red teaming LLM modelů a AI agentů na Microsoft Foundry. Skeny definované v YAML s PyRIT, garak a sadou nástrojů BlackIce, porovnatelné scorecardy s plným audit trailem a schvalovací brány, které model nebo agenta nepustí do nasazení, dokud neprojde."
tags: ["AI Red Teaming", "PyRIT", "garak", "BlackIce", "Microsoft Foundry", "GitHub Actions", "AI Security"]
keywords: ["ai red teaming", "bezpečnost llm", "pyrit", "garak", "blackice", "azure ai red teaming agent", "microsoft foundry", "prompt injection", "jailbreak", "bezpečnost ai agentů", "attack success rate", "schvalovací brána nasazení", "github actions", "devsecops pro ai"]
---

### Výzva

Stejný evropský výrobce baterií, který přistupuje k LLM přes mou [AI platformu na Microsoft Foundry](/cs/case-study/ai-foundry-platform-ai-gateway/), rychle nasazoval agenty: Terraform vývojáře, cloud security architekta, bezpečnostního analytika, Terraform auditora a interního asistenta — nad modely GPT-5 a Claude, s novými verzemi každých pár týdnů.

S každým vydáním přicházely stejné otázky a nikdo na ně neuměl odpovědět s důkazy:

- **Dá se agent přemluvit k něčemu, co dělat nesmí** — přímým jailbreakem, zakódovaným promptem nebo instrukcí schovanou v dokumentu, který čte?
- **Je verze 6 bezpečnější, nebo méně bezpečná než verze 5?** Ruční namátkové testy dávaly historky, ne čísla.
- **Kdo tento model schválil do produkce a na základě čeho?** Bezpečnostní review byla schůzka, ne artefakt.

Nástroje na red teaming existovaly, ale jako skripty na něčím notebooku: každý běh nastavený jinak, výsledky v lokální složce a nic, co by sken spojilo s konkrétním vydáním.

### Moje řešení

Red teaming jsem postavil jako krok pipeline: sken je definovaný v YAML, běží v CI proti konkrétnímu nasazení modelu nebo verzi agenta, vytvoří porovnatelný scorecard a rozhodne, jestli se ta verze smí nasadit.

{{< mermaid >}}
flowchart TB
    Y["📄 Profil skenu<br/>YAML v Gitu"]
    P["⚙️ GitHub Actions<br/>OIDC, bez secrets"]
    E["🎯 Útočné enginy<br/>PyRIT · garak · BlackIce"]
    T["🤖 Cíl<br/>nasazení modelu / verze agenta"]
    S["📊 Scorecard<br/>ASR podle kategorie a techniky"]
    G{"🚦 Schvalovací brána<br/>prahy"}
    D["🚀 Nasazení"]
    B["⛔ Zablokováno + report"]

    Y --> P --> E
    E -- útoky --> T
    T -- odpovědi --> E
    E --> S --> G
    G -- projde --> D
    G -- neprojde --> B
{{< /mermaid >}}

**Skeny jako konfigurace, ne kód.** Profil skenu je krátký YAML soubor: cíl (nasazení modelu nebo jméno a verze agenta), platforma, způsob autentizace, kategorie rizik, útočné strategie, počet cílů a kol a prahy, které rozhodují o úspěchu. Profily leží v Gitu vedle definic agentů, takže změna toho, co testujeme, prochází stejným review v pull requestu jako změna agenta samotného.

**Více enginů, jeden scorecard.** Základem je **PyRIT** přes Azure AI Red Teaming Agent, který pro každou kategorii rizik vygeneruje útočné cíle, spustí je a ohodnotí odpovědi. K tomu jsem přidal **garak** pro širokou sadu zranitelností modelů (prompt injection, triky s kódováním, únik dat, toxicita) a sadu nástrojů **BlackIce** — kontejnerizovaný red-teaming toolkit, který spojuje garak, PyRIT a další open-source skenery — pro hlubší běhy před velkými vydáními. Výstup každého enginu se převádí do stejného formátu scorecardu, takže výsledky jsou porovnatelné bez ohledu na to, který nástroj je vytvořil.

**Modely i agenti, lokálně i v cloudu.** Nasazení modelů se útočí přímo z GitHub Actions runneru, včetně nasazení Claude, která potřebují jiného klienta než ta kompatibilní s OpenAI. Agenti se skenují **cloud-native**: pipeline odešle úlohu na managed compute ve Foundry, Foundry spustí útoky proti přesné verzi agenta, vyhodnotí zakázané akce, únik citlivých dat a dodržení úkolu a uloží běh do portálu Foundry mezi Evaluations.

**Co se útočí.** Sedm kategorií rizik — násilí, sebepoškozování, sexuální obsah, nenávist a nespravedlnost, nepodložená tvrzení, zranitelnosti v kódu a u agentů zakázané akce — v kombinaci s 25 útočnými strategiemi: kódování a šifry (Base64, ROT13, Morseovka, zaměnitelné Unicode znaky, ASCII smuggling), přímé jailbreaky, **nepřímý prompt injection** přes dokumenty a kontext a vícekolová eskalace jako Crescendo.

**Auditovatelné a porovnatelné.** Každý běh nese v názvu cíl, jeho verzi, profil skenu a ID běhu GitHub Actions a odkazuje přímo na evaluaci ve Foundry. Scorecard — Attack Success Rate podle kategorie rizik a útočné techniky — se zapisuje do souhrnu jobu a ukládá jako HTML report. Protože jsou skeny verzované a opakovatelné, porovnává se verze 6 agenta s verzí 5 na stejném profilu, ne na jiné sadě promptů. Surové výstupy s vygenerovaným škodlivým obsahem se drží jen jako krátkodobé artefakty a nikdy se necommitují.

**Schvalovací brány pro nasazení.** Scorecard není jen report. Pipeline pro nasazení modelu nebo agenta zavolá red-team krok a porovná výsledek s prahy v profilu; verze, která v kterékoli kategorii překročí povolený Attack Success Rate, se nenasadí. Produkční nasazení navíc čeká na povinného schvalovatele, který schvaluje se scorecardem před sebou. Týdenní plánovaný sken znovu testuje to, co už v produkci běží, protože model pod agentem se může změnit, aniž by na agenta kdokoli sáhl.

**Poctivě k hodnotiteli.** Automatičtí hodnotitelé jsou na úrovni jednotlivé položky nepřesní: občas označí odmítnutí za úspěšný útok a naopak. Pipeline proto přebírá počty prošlých a neprošlých položek přímo od evaluátorů, souhrnný ASR bere jako trend a před citováním čísla mimo tým vyžaduje ruční kontrolu vzorku neúspěšných položek.

### Dopad

{{< impact-grid cols="4" >}}
25|Útočných strategií|red
7|Kategorií rizik|purple
Každá verze|Skenovaná před nasazením|blue
100 %|Běhů dohledatelných k vydání|green
{{< /impact-grid >}}

### Klíčové výsledky

- **Bezpečnostní review se stalo artefaktem**: každá verze modelu a agenta jdoucí do produkce má scorecard, ID běhu a jmenovaného schvalovatele.
- **Regrese jsou vidět**: stejný YAML profil proti dvěma verzím ukáže přesně, která kategorie nebo technika se zhoršila.
- **Jeden proces pro modely i agenty**: nasazení GPT-5 i Claude, nextgen i klasičtí agenti, skeny na runneru i cloud-native končí ve stejném scorecardu.
- **Žádné secrets**: pipeline se přihlašuje přes OIDC workload identity federation; klíče jsou jen volba pro cíle, které je vyžadují.
- **Připraveno na rozšíření**: nové cíle jako Databricks model-serving endpointy se připojí přes callback a využijí stejné hodnocení i brány.
