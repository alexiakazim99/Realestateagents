---
name: done-check
description: >
  Läs och följ denna skill innan du säger att något är klart — en agent, ett
  workflow, en bugfix, en ändring. "Klart" betyder testat, dokumenterat och
  sparat, inte bara byggt. Bocka av alla punkter innan du rapporterar något
  som färdigt.
---

# Done-check – innan något får kallas klart

Säg inte att något är klart förrän varje punkt nedan är avbockad. Om en punkt
inte går att bocka av, säg det rakt ut istället för att kalla arbetet klart.

## 1. Testa att det funkar
- Kör flödet med riktig testdata — anta inte att det funkar för att det ser rätt ut.
- Följ hela vägen från trigger till slutresultat (mejl skickat, rad skriven,
  svar returnerat).

## 2. Testa specialfallen
- Vad händer vid tom input, saknad fil, ingen träff i sheetet, dubblett, eller
  när något externt inte svarar?
- Kör minst det viktigaste specialfallet, inte bara det lyckade fallet.

## 3. Uppdatera dokumentationen
- Ändra `architecture`-skillen och relevanta `docs/`-filer om något ändrats
  (flöde, routing, kopplingar).
- Lägg en rad i ändringsloggen: vad ändrades och varför.

## 4. Spara arbetet
- Exportera workflowet från n8n som JSON till `workflows/`.
- Håll agentnamn och filnamn konsekventa.

## 5. Committa rent
- Ett begripligt commit-meddelande som säger vad som gjordes.
- **Inga hemligheter i diffen** — inga API-nycklar, webhook-secrets, ngrok-URL:er
  eller credentials. Kontrollera diffen innan commit.
- Sammanfatta vilka filer som ändrats så användaren kan granska före commit.

## Rapportera ärligt
När du är klar, säg vad du testade och vad du *inte* testade. Om något är osäkert
eller ogjort — säg det. Ett halvtestat "klart" är värre än ett ärligt "det här
återstår".
