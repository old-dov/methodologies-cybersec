# Rapport Méthodologique — Pentest Grey Box API REST IBC-News

**Type d'engagement :** Grey Box Penetration Test — API REST
**Référentiel :** OWASP API Security Top 10 (2023)
**Cible :** Plateforme de publication IBC-News (IBC Corporation)
**Périmètre :** Frontend React SPA (`10.10.7.50:4100`), Backend API Node/Express (`10.10.7.100:8000`), infrastructure réseau associée
**Outils utilisés :** navigateur (DevTools), curl, grep, nmap, redis-cli, script Python custom (password spraying)

---

## 1. Contexte et objectifs

IBC-News a migré sa plateforme de publication vers une architecture REST API en vue du lancement de ses applications mobiles. L'engagement visait à évaluer la robustesse de cette API contre l'OWASP API Security Top 10, avec un accès Grey Box : accès à l'application frontend et à la documentation Postman officielle, sans accès direct à l'infrastructure serveur (pas de SSH).

**Prémisse de l'engagement :** un jeu de données de fuite (`leaked_passwords.txt`) avait été intercepté par la threat intel, sans emails associés — la première phase de reconnaissance visait donc à identifier des emails d'employés valides exploitables pour du credential stuffing / password spraying.

---

## 2. Méthodologie générale

L'engagement a suivi une approche en 3 phases classiques d'un pentest API :

1. **Reconnaissance & cartographie de la surface d'attaque**
   - Cartographie de la collection Postman officielle (documentation fournie) : 6 catégories fonctionnelles, 33 endpoints documentés (Auth, Articles, Articles/Favorite/Comments, Profiles, Tags, Membership).
   - Analyse du trafic réseau (DevTools → Network) pour identifier l'architecture réelle : séparation frontend (`:4100`, Express + React) / backend API (`:8000`).
   - Analyse statique du bundle JavaScript compilé du frontend (`/static/js/bundle.js`, ~2.9 Mo) via `grep` pour extraire les routes non documentées codées en dur dans les définitions de `<Route>` React Router.

2. **Exploitation ciblée, un vecteur OWASP à la fois**
   - Chaque route/comportement suspect identifié en phase 1 a été testé isolément via `curl` en manipulant : méthode HTTP, en-têtes d'authentification, structure du body JSON, paramètres de requête (query params), et version de chemin d'API (`/v1/` vs `/v2/`).
   - Authentification obtenue par password spraying (voir §3.3), permettant de rejouer les tests avec des tokens JWT valides sur des comptes non privilégiés — condition nécessaire pour distinguer les failles d'autorisation "verticales" (rôle) des failles "horizontales" (BOLA/IDOR).

3. **Pivot réseau et service discovery**
   - Une fois la surface applicative épuisée, scan du sous-réseau complet (`nmap -sn` puis `nmap -p- --min-rate=1000`) pour cartographier les services annexes exposés sur l'infrastructure (bases de données, caches), au-delà des deux hôtes frontend/backend déjà identifiés.

---

## 3. Détail des findings (mappés OWASP API Security Top 10:2023)

### 3.1 — API5:2023 Broken Function Level Authorization
**Endpoint :** `GET /api/admin` (backend), route frontend `/admin` non documentée dans Postman
**Découverte :** extraction de routes codées en dur dans `bundle.js` via `grep -oE '"/[a-zA-Z0-9/_-]*"'` — la route `/admin` est apparue dans la liste des chemins React Router, absente de toute documentation officielle. Le composant associé (`AdminPage`) est chargé en lazy-loading dans le chunk `SettingsScreen.chunk.js`, regroupé avec un composant fonctionnellement sans rapport — ce qui masque partiellement sa présence lors d'une revue de code superficielle.
**Exploitation :** appel direct à l'endpoint API backend correspondant avec un token JWT valide appartenant à un compte standard (`admin: false`). Aucune vérification de rôle n'était appliquée côté serveur — seule la présence d'un token valide était contrôlée (403 sans token, 200 avec n'importe quel token valide).
**Impact métier :** exposition de la liste complète des emails employés à tout utilisateur authentifié, sans besoin de privilège administrateur. Sert de point d'entrée à des attaques ultérieures (password spraying ciblé, phishing).

### 3.2 — API1:2023 Broken Object Level Authorization (BOLA)
**Endpoint :** `PUT /api/articles/{slug}` vs `PUT /api/articles/{id}`
**Découverte :** la collection Postman documente l'accès aux articles via `slug`. Un test de modification via le `slug` d'un article appartenant à un autre utilisateur a correctement échoué (403 — "you are not an author of this article"), indiquant une vérification de propriété fonctionnelle sur ce chemin.
**Exploitation :** le même contrôle n'était **pas** répliqué sur le chemin alternatif utilisant l'ID numérique interne de l'article (`PUT /api/articles/{id}`) — vecteur non documenté dans Postman, découvert par déduction (l'API expose l'`id` numérique dans les réponses `GET /articles`). La modification a réussi sans aucune vérification de propriété.
**Impact métier :** tout utilisateur authentifié peut altérer ou supprimer le contenu éditorial de n'importe quel autre auteur, avec un risque de désinformation financière sur une plateforme de news marché — impact réputationnel et potentiellement réglementaire significatif pour ce secteur d'activité.

### 3.3 — API2:2023 Broken Authentication
**Endpoint :** `POST /api/v2/users/login`
**Découverte :** confrontation d'une liste de mots de passe divulgués (`leaked_passwords.txt`, 99 entrées) contre les 6 adresses email obtenues via le finding §3.1, par attaque de type **password spraying** (un mot de passe testé sur l'ensemble des comptes avant de passer au suivant, plutôt qu'un bruteforce croisé complet — approche plus discrète et représentative d'une attaque réelle).
**Exploitation :** script Python automatisé (`requests`), boucle externe sur les mots de passe / interne sur les emails, détection de succès sur code HTTP 200.
**Résultat :** **6 comptes sur 6 compromis** (taux de compromission de 100%), avec des mots de passe issus du top des listes de mots de passe les plus communs au monde (`iloveyou`, `123456789`, `1q2w3e4r`...).
**Impact métier :** absence totale de politique de complexité de mot de passe et absence de mécanisme de limitation du nombre de tentatives (rate limiting) sur l'endpoint de login — a permis un test exhaustif sans blocage ni ralentissement.

### 3.4 — API3:2023 Broken Object Property Level Authorization / Excessive Data Exposure
**Endpoint :** `GET /api/profiles/{username}`
**Découverte :** l'interface utilisateur n'affiche que des données publiques de profil (bio, avatar). L'inspection directe de la réponse JSON brute de l'API (avec un token d'un compte tiers, non propriétaire du profil) a révélé des champs additionnels non filtrés par le backend avant sérialisation.
**Exploitation :** appel direct de l'endpoint avec un compte compromis (§3.3), récupération de données financières complètes d'un autre utilisateur (nom sur la carte, numéro complet, CVC, date d'expiration).
**Démonstration d'impact ("use", pas seulement "extract") :** les données de carte extraites ont été soumises avec succès à l'endpoint `POST /api/membership` sous l'identité d'un compte tiers, démontrant l'exploitabilité financière concrète de la fuite (pas uniquement théorique).
**Impact métier :** exposition de données de paiement (PCI-DSS scope) à l'ensemble des utilisateurs authentifiés de la plateforme — l'un des findings les plus critiques de l'engagement.

### 3.5 — API9:2023 Improper Inventory Management
**Endpoint :** `POST /api/v1/users/login`
**Découverte :** la documentation Postman officielle référence exclusivement des endpoints préfixés `/v2/`. Hypothèse testée : existence résiduelle d'une version `/v1/` non dépréciée fonctionnellement.
**Exploitation :** requête directe sur le chemin `/v1/` équivalent — endpoint toujours actif et fonctionnel, en parallèle de la version documentée.
**Impact métier :** surface d'attaque non inventoriée / non surveillée ; une version antérieure peut contenir des protections de sécurité moins abouties que la version courante (hypothèse à vérifier en profondeur si l'engagement se poursuit), et échappe par nature aux contrôles de sécurité appliqués sciemment sur les endpoints connus.

### 3.6 — API4:2023 Unrestricted Resource Consumption
**Endpoint :** `GET /api/articles?limit=&offset=`
**Découverte :** les paramètres de pagination documentés (`limit`, `offset`) ne présentaient aucune borne supérieure visible dans la documentation.
**Exploitation :** envoi d'une valeur de `limit` extrême (near-illimitée) — provoque un crash serveur (HTTP 500), le message d'erreur confirmant explicitement une "allocation de ressources excessive".
**Impact métier :** déni de service trivialement déclenchable par un utilisateur non privilégié, sans nécessiter de volume de requêtes (single request DoS) — risque de disponibilité pour l'ensemble de la plateforme.

### 3.7 — API3:2023 Mass Assignment
**Endpoint :** `PUT /api/user`
**Découverte :** le schéma documenté dans Postman pour la mise à jour de profil ne référence que le champ `email`. Hypothèse : absence de filtrage des propriétés côté serveur (binding aveugle du body JSON au modèle de données).
**Exploitation :** ajout d'une propriété non documentée et non prévue (`admin`) dans le body de la requête, avec une valeur usurpée.
**Résultat :** élévation de privilège persistante confirmée — le compte de test est passé de `admin: false` à `admin: true` en base, avec émission d'un nouveau token JWT reflétant ce nouveau statut.
**Impact métier :** élévation de privilèges triviale et persistante pour tout utilisateur authentifié, sans nécessiter de vulnérabilité applicative complexe — un seul champ JSON ajouté suffit.

### 3.8 — API8:2023 Security Misconfiguration / Injection
**Endpoint :** `POST /api/debug` (microservice FastAPI distinct, non documenté dans la collection Postman)
**Découverte :** comparaison entre la documentation Postman et une interface Swagger UI trouvée sur une route non référencée (`/docs`), exposant la spécification OpenAPI complète (`/openapi.json`) — révélant un endpoint `/api/debug` totalement absent de toute documentation officielle.
**Exploitation :**
  - Analyse du schéma OpenAPI pour déterminer la structure exacte attendue du payload (`{"body": {"command": "..."}}`).
  - Premier test bloqué par une whitelist de commandes autorisées (validation côté applicatif).
  - Contournement de la whitelist par chaînage de commandes shell (opérateur de séparation de commandes), permettant l'exécution d'une commande arbitraire non whitelistée en l'accolant à une commande autorisée.
**Impact métier :** exécution de code arbitraire côté serveur (RCE) — impact maximal, compromission totale du système hôte du microservice concerné.

### 3.9 — API7:2023 Server Side Request Forgery / Security Misconfiguration
**Service :** instance Redis exposée sur le réseau interne du lab
**Découverte :** scan de découverte d'hôtes (`nmap -sn`) sur la plage réseau complète contenant les IPs frontend et backend connues, révélant deux hôtes supplémentaires non identifiés jusque-là. Scan de ports complet (`nmap -p-`) sur ces hôtes révélant un service PostgreSQL sur l'un et un service Redis sur l'autre, tous deux sur leurs ports par défaut.
**Exploitation :** connexion au service Redis via `redis-cli` sans authentification — accès accepté sans aucune credential. Énumération des clés stockées (`KEYS *`) révélant l'exposition directe du contenu du datastore.
**Impact métier :** un service de cache/datastore interne, censé n'être accessible que par les composants applicatifs backend, est directement joignable et manipulable depuis n'importe quel point du réseau — absence de segmentation réseau et absence d'authentification sur un composant d'infrastructure critique.

---

## 4. Synthèse des résultats

| # | Vulnérabilité | Catégorie OWASP API Top 10:2023 | Sévérité indicative |
|---|---|---|---|
| 1 | Route admin non protégée par rôle | API5 — Broken Function Level Authorization | Élevée |
| 2 | Modification d'article via ID sans contrôle de propriété | API1 — Broken Object Level Authorization | Élevée |
| 3 | Compromission de 6/6 comptes par mots de passe faibles | API2 — Broken Authentication | Critique |
| 4 | Exposition de données de carte bancaire via profil | API3 — Excessive Data Exposure | Critique |
| 5 | Endpoint de login v1 obsolète toujours actif | API9 — Improper Inventory Management | Moyenne |
| 6 | Absence de limite sur la pagination → crash serveur | API4 — Unrestricted Resource Consumption | Élevée |
| 7 | Élévation de privilège par mass assignment | API3 — Mass Assignment | Critique |
| 8 | Exécution de commande arbitraire via endpoint non documenté | API8 — Injection / Security Misconfiguration | Critique |
| 9 | Datastore Redis exposé sans authentification | Security Misconfiguration | Élevée |

**Taux de compromission des comptes testés :** 100% (6/6) via attaque par dictionnaire de mots de passe communs.

**Constat transverse :** l'ensemble des findings partage une cause racine commune — une confiance excessive placée dans le frontend comme couche de contrôle d'accès et de filtrage, le backend appliquant des vérifications incomplètes, incohérentes selon les chemins d'accès à une même ressource (slug vs id), ou totalement absentes sur les endpoints non destinés à un usage grand public (`/admin`, `/debug`, microservices annexes).

---

## 5. Recommandations générales

1. **Centraliser l'autorisation côté serveur** — implémenter un middleware de contrôle d'accès systématique, appliqué uniformément à tous les chemins d'accès à une ressource (peu importe l'identifiant utilisé : slug, id, ou autre), plutôt que des vérifications ad hoc dispersées dans le code.
2. **Adopter une politique de mots de passe robuste** côté serveur (longueur minimale, vérification contre des listes de mots de passe compromis type HaveIBeenPwned, MFA pour les comptes à privilèges) et un rate limiting sur les endpoints d'authentification.
3. **Filtrer les données en sortie (output filtering)** par un schéma de sérialisation explicite (whitelist de champs autorisés par contexte d'appel), plutôt que de renvoyer l'objet de données complet du modèle interne.
4. **Filtrer les données en entrée (mass assignment)** — lier explicitement uniquement les champs attendus d'un payload à un modèle de données, jamais par binding automatique complet.
5. **Établir un inventaire d'API exhaustif et maintenu** (gestion de version stricte avec dépréciation effective des anciennes versions, découverte automatisée des endpoints exposés, revue régulière des microservices annexes).
6. **Appliquer des limites de ressources systématiques** sur tout paramètre influençant le volume de données traité ou renvoyé.
7. **Ne jamais exposer d'endpoints d'exécution de commande**, même protégés par whitelist — un filtrage par liste blanche de commandes reste contournable par les métacaractères shell ; privilégier des architectures ne nécessitant aucune exécution de commande dynamique pilotée par l'utilisateur.
8. **Segmenter le réseau et exiger une authentification** sur tout composant d'infrastructure interne (bases de données, caches), même en environnement supposé non exposé publiquement.

---

*Rapport rédigé dans le cadre du module Jedha Cyber Lead — méthodologie réutilisable pour engagements Grey Box API REST.*
