---
name: lead-agent
description: >
  Läs denna skill innan du ändrar Lead Agent. Den tekniska nod-för-nod-
  beskrivningen finns i docs/lead-agent.md — läs den för att förstå flödet.
  Denna fil samlar bara vad du måste tänka på när du rör agenten, så inget
  går sönder.
---

# Lead Agent – att tänka på vid ändringar

Teknisk beskrivning: se `docs/lead-agent.md`. Denna fil är beteenderegler, inte
en flödesbeskrivning.

## Innan du ändrar
- Läs `docs/lead-agent.md` först så du förstår båda kedjorna (webhook-kedjan och
  den schemalagda formulärlänk-kedjan). De är oberoende — ändra inte den ena i
  tron att den påverkar den andra.

## Testa alltid detta
- **Både träff och miss.** När ett objekt matchar en mäklare *och* när det inte
  gör det (fallback till admin). Missfallet glöms lätt och är minst lika viktigt.
- **Loggningen.** Varje lead ska hamna som en rad i Leads-fliken, matchad eller ej.

## Rör inte utan att testa outputen
- **Categorize-nodens JSON.** Agenten förväntar sig strikt JSON (temperatur,
  prioritet, motivering). Ändrar du prompten, verifiera att outputen fortfarande
  går att tolka — annars kraschar stegen efter.
- **Matchningen mot sheetet.** Objekt matchas normaliserat (trim + gemener).
  Ändrar du kolumnnamn eller matchningslogik, testa att en känd rad fortfarande
  hittas.

## Känsligt
- **Inga hemligheter** i prompt, workflow-export eller commit (webhook-nyckeln
  bl.a.).
- Följ `done-check` innan du kallar en ändring klar.
