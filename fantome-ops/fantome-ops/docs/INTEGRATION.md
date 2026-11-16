# Contrat avec les parties Discord

Ce que les composants Discord (à écrire) doivent respecter pour que la chaîne fonctionne.

## Étape 1 — export Discord
- Un message affiche une date « leurre » (ex. 3 mars) alors que son **snowflake décode en `KEY_DATE`**
  (UTC, `(id >> 22) + 1420070400000` ms). `KEY_DATE` est dans `secrets.env`.
- Le statut personnalisé de Léo (dans `activity/`) = contenu exact de `out/step2_status.txt`
  (ou `out/step2_status.json`). Il doit sembler vide.
- Leurres : faux flag et faux mot de passe, sans lien avec `STEGO_PASSWORD`.

## Étape 3 — Sentinelle
- `ID du bot` = `SENTINELLE_BOT_ID` (à renseigner dans `secrets.env` **puis** relancer `make step2`).
- `/audit <id_message>` : vérifie l'appartenance au serveur mais pas l'accès au canal (faille voulue).
  Version durcie : jeton `HMAC-SHA256(STEGO_PASSWORD, "audit:<user_id>")[:32]`.
- Le message de `#archives-ops` contient le lien de `out/sentinelle.tar`.
- `/ping <service> <token>` relaie **tel quel** (aucun nettoyage côté bot) :

```python
r = httpx.get(f"{OPS_API_URL}/ping", params={"service": service},
              headers={"Authorization": f"Bearer {token}"}, timeout=10)
await interaction.response.send_message(f"```json\n{r.text[:1500]}\n```")
```
- Les secrets de Sentinelle (`DISCORD_TOKEN`, etc.) arrivent par variables d'environnement au runtime.
  Le `.env` embarqué dans l'image ne contient que `SIGNING_KEY`, `OPS_API_URL`, `STEP4_FLAG`.
- Remplace `step4-docker/bot/main.py` par le vrai bot, puis `make image`.

## Étape 6 — Archiviste (colle Discord)
```python
msgs = [{"author": m.author.display_name, "content": m.content}
        async for m in sandbox.history(limit=30)]
r = httpx.post("http://archiviste-llm:8081/summarize",
               json={"channel": "sandbox", "messages": msgs[::-1]},
               headers={"X-Team": team_id}, timeout=90)
await interaction.followup.send(r.json()["summary"])
```
- Le bot ne doit PAS accepter de texte libre dans `/resume` (sinon pas d'injection indirecte).
- Une instance/une catégorie par équipe : le quota (`MAX_REQ_PER_TEAM`) se compte par `X-Team`.

## Assemblage
`FLAG = PREFIX{ HALF_1 _ HALF_2 }` — pas de chiffrement final : la protection est la chaîne.
Flags intermédiaires : `STEP2_FLAG` (payload stégano), `STEP4_FLAG` (.env de l'image),
`STEP5_FLAG` (réponse de `/internal/keyshard`). Étapes 1, 3 et 6 : à toi de définir/poser leur flag
(étape 6 = flag final).
