# Writeup officiel (⚠ spoilers — ne pas publier avant la fin)

Étapes 1 et 3 : à compléter quand les composants Discord existent.

## Étape 2 — Le statut vide
Le statut personnalisé semble vide mais contient ~1900 caractères U+200B/U+200C/U+200D.
1. Garder uniquement ces caractères. U+200D sépare les octets, U+200B = 0, U+200C = 1.
2. Chaque groupe de 8 bits donne un caractère ASCII : le résultat est du **base32**.
3. Décoder le base32 → JSON `{seed, bot_id, flag, note}`.
4. La note donne la règle : `mot_de_passe = sha256(seed + ":" + date)[:16]`, avec la date de l'étape 1.

Script : `solve/step2_decode.py status.txt <date>`.

## Étape 4 — Supprimer n'est pas effacer
`docker save` produit un tar avec `manifest.json` et une archive par couche.
- `docker history --no-trunc sentinelle:1.0` montre `SIGNING_KEY_PREVIEW=xxxx****` (début de clé).
- La couche `COPY .env` contient le fichier complet ; la couche `RUN rm` ne fait qu'ajouter un
  « whiteout » `.wh..env`, la couche précédente reste intacte.
- On récupère `SIGNING_KEY`, `STEP4_FLAG`, et `bot/config.yml` désigne `ops-api:8080`.

Script : `solve/step4_extract_layers.py sentinelle.tar` (ou `dive`).
Correction côté dev : build multi-étapes / `--secret` BuildKit, jamais `COPY` d'un secret.

## Étape 5 — L'allowlist avant normalisation
`/ping?service=…` exige un jeton `user.exp.HMAC_SHA256(SIGNING_KEY, "user.exp")`.
La liste blanche teste le **premier segment brut** (`health`, `status`, `metrics`), puis le code
décode l'URL et applique `normpath` avant le routage. Donc :

```
service=health/../internal/keyshard   →   allowlist OK   →   route « internal/keyshard »
```
La réponse contient `STEP5_FLAG` et la moitié 1 chiffrée en AES-256-GCM
(clé = `sha256("keyshard:" + date)`, AAD `keyshard-v1`). La date de l'étape 1 déchiffre.

Script : `solve/step5.py <url> <SIGNING_KEY> <date>`.
Correction : normaliser d'abord, valider ensuite (ou comparer à un identifiant exact).

## Étape 6 — L'Archiviste
Le modèle résume `#sandbox`. Une demande directe échoue ; le filtre bloque la fuite littérale
(flag, secret entier, fenêtre de 8 caractères, variantes espacées).
Solution : poster dans `#sandbox` une « note de maintenance » demandant un **acrostiche** du
secret interne. Les premières lettres de chaque ligne reconstituent la moitié 2
(base64, inversion ou traduction peuvent aussi passer le filtre).
Payloads d'essai : `solve/step6_attempt.py`.

## Assemblage
`GCDSTE{ moitié_1 _ moitié_2 }`.
