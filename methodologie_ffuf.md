# Méthodologie de Fuzzing Web avec ffuf

**Contexte :** audit de sécurité de la plateforme Ariake Technologies (`localhost:5000`)
**Outil :** ffuf v2.1.0
**Wordlists :** SecLists — `Discovery/Web-Content/`

---

## Principe général

ffuf remplace le mot-clé `FUZZ` dans une URL, un paramètre ou un header par chaque entrée d'une wordlist, puis observe les réponses (code HTTP, taille, mots, lignes) pour identifier ce qui existe réellement sur le serveur.

Commande de base :
```bash
ffuf -u http://cible/FUZZ -w wordlist.txt [options]
```

Options clés utilisées tout au long de l'audit :

| Option | Rôle |
|---|---|
| `-w` | Chemin de la wordlist |
| `-e` | Extensions à tester en plus du mot brut (`.php,.zip,.bak`...) |
| `-mc` | *Match codes* — codes HTTP à afficher |
| `-fc` | *Filter codes* — codes HTTP à masquer |
| `-fs` | *Filter size* — masque les réponses d'une taille donnée (bruit de fond) |
| `-c` | Sortie colorée |
| `-t` | Nombre de threads |

---

## Étape 1 — Découverte de répertoires

**Objectif :** identifier les dossiers exposés sur le serveur.

```bash
ffuf -u http://localhost:5000/FUZZ \
  -w ~/SecLists/Discovery/Web-Content/raft-medium-directories.txt \
  -mc 200,204,301,302,307,401,403,405 \
  -c
```

**Logique du choix des codes (`-mc`) :** on ne se limite pas au 200. Un 401/403 signale un endpoint qui *existe* mais qui est protégé — information précieuse même sans accès direct.

**Résultat :** `/search` et `/console` détectés.

---

## Étape 2 — Découverte de fichiers

**Objectif :** trouver des fichiers isolés (configs, sources, archives) que le scan de répertoires ne révèle pas, car ils ne créent pas de listing.

```bash
ffuf -u http://localhost:5000/FUZZ \
  -w ~/SecLists/Discovery/Web-Content/raft-medium-files.txt \
  -mc 200 -fc 404 \
  -c
```

**Résultat :** `robots.txt` découvert. Sa lecture (`curl`) révèle des chemins supplémentaires non listés par ffuf lui-même :
```
Disallow: /admin/
Disallow: /backup/
Disallow: /api/
Disallow: /.git/
```

> **Point méthodologique :** `robots.txt` n'est pas un résultat de fuzzing — c'est une fuite d'information exploitée *après* la découverte. Toujours le lire dès qu'il apparaît : il donne souvent des cibles gratuites pour la suite du fuzzing.

---

## Étape 3 — Fuzzing ciblé des sous-répertoires révélés

Chaque chemin de `robots.txt` doit être vérifié individuellement (avec `curl -i` pour voir le code HTTP), puis fuzzé s'il contient potentiellement d'autres ressources.

**Exemple sur `/backup/` :** un répertoire peut renvoyer 404 sur son index tout en contenant des fichiers accessibles individuellement. Il faut donc fuzzer *dans* le dossier, pas seulement tester le dossier lui-même :

```bash
ffuf -u http://localhost:5000/backup/FUZZ \
  -w ~/SecLists/Discovery/Web-Content/raft-medium-files.txt \
  -e .zip,.txt,.bak,.sql,.config,.md \
  -mc 200 -fc 404 \
  -c
```

**Erreur commise durant l'audit :** on s'est arrêté après avoir trouvé `source.zip` sans creuser plus loin, alors que le dossier contenait aussi `config.txt` et `notes.txt` — deux fichiers portant chacun un indice/flag. **Leçon : sur un dossier fuzzé avec succès, toujours laisser le scan aller au bout et examiner *tous* les résultats, pas seulement le premier qui semble prometteur.**

---

## Étape 4 — Fuzzing de paramètres GET

**Objectif :** détecter des paramètres cachés qui changent le comportement de l'application (modes debug, filtres non documentés, etc.), sur une page qui répond normalement en 200 sans paramètre particulier.

### 4.1 Établir la baseline

Avant de fuzzer, il faut connaître la taille de réponse "normale" (sans paramètre valide), pour pouvoir la filtrer ensuite :

```bash
curl -s -o /dev/null -w "%{size_download}\n" "http://localhost:5000/?zzznonexistent=1"
```

### 4.2 Lancer le fuzzing avec filtrage par taille

```bash
ffuf -u "http://localhost:5000/?FUZZ=true" \
  -w ~/SecLists/Discovery/Web-Content/raft-medium-words.txt \
  -mc 200 \
  -fs <TAILLE_BASELINE> \
  -c
```

**Principe du filtre `-fs` :** par défaut, une application ignore les paramètres qu'elle ne reconnaît pas et renvoie toujours la même page. En filtrant cette taille récurrente, seuls les paramètres qui *changent réellement* la réponse restent visibles dans les résultats.

> **Erreur commise durant l'audit :** la première tentative de ce fuzzing a été faite avec une wordlist mal calibrée sur `/search?q=...` (paramètre déjà connu) plutôt que sur la racine `/`, et avec une valeur de `-fs` mal vérifiée. Résultat : aucun filtrage effectif, des milliers de faux positifs. **Leçon : toujours vérifier manuellement (`curl`) la taille de baseline juste avant de lancer ffuf, sur l'endpoint exact qu'on va fuzzer** — une baseline copiée d'un autre test ou approximative rend le filtre inutile.

**Résultat attendu :** un paramètre comme `debug=true` sort du lot (taille de réponse différente), révélant un mode debug caché avec des informations sensibles.

---

## Étape 5 — Fuzzing de l'API

**Objectif :** cartographier les endpoints d'une API découverte (ex. via `robots.txt` → `/api/`).

```bash
ffuf -u http://localhost:5000/api/FUZZ \
  -w ~/SecLists/Discovery/Web-Content/raft-medium-words.txt \
  -mc 200,201,204,301,302,307,400,401,403,405 \
  -c
```

**Points clés :**
- Un `401` (`api_key required`) indique un endpoint réel mais protégé par une clé — pas une impasse, une piste à exploiter (ex. injection SQL pour extraire la clé depuis une base de données).
- Un `405` (Method Not Allowed) signale un endpoint qui existe mais attend une autre méthode HTTP (souvent POST) — à retester avec `curl -X POST`.
- Une wordlist générique dédiée aux endpoints API (`api/api-endpoints.txt`) peut échouer si l'application utilise des noms métier courants (`upload`, `status`...). Dans ce cas, basculer sur une wordlist de mots génériques (`raft-medium-words.txt`) est plus efficace.

---

## Résumé du déroulé complet de l'audit

| Ordre | Action | Outil | Résultat |
|---|---|---|---|
| 1 | Fuzzing répertoires racine | ffuf | `/search`, `/console` |
| 2 | Fuzzing fichiers racine | ffuf | `robots.txt` → chemins `/admin/`, `/backup/`, `/api/`, `/.git/` |
| 3 | Fuzzing dans `/backup/` avec extensions | ffuf | `source.zip`, `config.txt`, `notes.txt` (flags/indices) |
| 4 | Lecture `.git/config` | curl | Flag (URL du repo + commentaire) |
| 5 | Fuzzing paramètres sur `/` avec baseline calibrée | ffuf | Paramètre `debug=true` → flag |
| 6 | Fuzzing `/api/FUZZ` | ffuf | `/api/status`, `/api/upload` |
| 7 | Injection SQL sur `/search?q=` | curl (UNION-based) | Extraction clé API |
| 8 | Upload du fichier malveillant via `/api/upload` | curl -X POST | Crash confirmé + flag final |

---

## Leçons méthodologiques clés

1. **Chaque fichier trouvé mérite d'être lu en entier**, pas juste identifié — les indices textuels (commentaires, notes) valent souvent plus que le fichier lui-même.
2. **Un dossier en 404 sur son index n'est pas forcément vide** — fuzzer systématiquement son contenu.
3. **Le fuzzing de paramètres nécessite une baseline vérifiée manuellement**, jamais supposée ou recyclée d'un autre endpoint.
4. **Un code 401/403/405 est une piste, pas une impasse** — il confirme l'existence de l'endpoint et indique souvent ce qu'il attend (clé, méthode).
5. **Ne pas confondre "endpoint verrouillé nécessitant un effort disproportionné" (ex. console Werkzeug avec PIN système) et "piste à explorer"** — savoir identifier les vrais dead ends fait partie de l'exercice.
