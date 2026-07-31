# Méthodologie d'Investigation Réseau : Analyse d'une Attaque Web via Wireshark (Cas BookWorld)

Cette fiche documente la méthodologie d'analyse d'une capture réseau (`WebInvestigation.pcap`) via Wireshark (interface graphique), dans le cadre de l'investigation d'une attaque web multi-étapes sur un serveur e-commerce : injection SQL, brute-force du panneau d'administration, puis upload d'un webshell.

#### 0. Vue d'ensemble du trafic

**Statistics → Protocol Hierarchy**
Donne la répartition des protocoles présents dans la capture (volumétrie par protocole/sous-protocole).

**Statistics → Conversations → onglet IPv4**
Liste les paires d'IP en communication, triable par nombre de paquets/octets (clic sur l'en-tête de colonne "Frames"). Permet d'identifier rapidement l'IP la plus active du trafic.

#### 1. Identification de l'IP attaquante

**Statistics → Conversations → IPv4**, tri décroissant sur la colonne "Frames".

Filtre de confirmation :
```
ip.addr == <IP_CANDIDATE> and http.request
```

#### 2. Géolocalisation de l'IP attaquante

Wireshark ne fait pas de géolocalisation native. Une fois l'IP identifiée à l'étape précédente, sortir de Wireshark et effectuer une recherche via un service externe de géolocalisation IP (whois, GeoIP lookup).

#### 3. Identification du script PHP vulnérable

Filtre :
```
http.request and ip.src == <IP_ATTAQUANT>
```

Ajout d'une colonne personnalisée pour visualiser les URI en liste :

- Clic droit sur un paquet HTTP → champ `Hypertext Transfer Protocol` → `Request URI` → **Apply as Column**

Tri chronologique (colonne Time par défaut) et lecture visuelle des URI pour repérer le script ciblé de façon répétée avec des payloads suspects dans ses paramètres.

#### 4. Première requête d'injection SQL (SQLi)

Filtre pour isoler les requêtes contenant une syntaxe d'injection :
```
http.request.uri contains "search=" and (http.request.uri contains "27" or http.request.uri contains "OR" or http.request.uri contains "AND")
```

- Tri chronologique croissant.
- Repérer la première occurrence contenant une syntaxe SQL structurée (opérateur logique + commentaire SQL).
- Clic droit sur le paquet → **Copy → Value** pour extraire l'URI brute (encodée URL).
- Décodage : coller la valeur extraite dans un outil de décodage URL externe (ex. CyberChef, ou tout décodeur `%XX` → caractère).

#### 5. Requête d'énumération des bases de données

Filtre :
```
http.request.uri contains "SCHEMATA" or http.request.uri contains "schema_name"
```

Même procédure de copie/décodage que l'étape précédente sur le résultat obtenu.

#### 6. Identification de la table contenant les données utilisateurs

Filtre pour isoler les requêtes d'énumération de structure de base de données :
```
http.request.uri contains "information_schema.tables" or http.request.uri contains "information_schema.columns"
```

- Suivre la séquence chronologique des requêtes (typique d'un outil d'exploitation automatisé type sqlmap) : énumération des tables, puis énumération des colonnes de la table ciblée.
- Le nom de table apparaît en clair dans les URI des requêtes suivantes qui la ciblent spécifiquement.

#### 7. Répertoire caché découvert

Filtre pour isoler les réponses positives sur les chemins scannés :
```
ip.src == <IP_ATTAQUANT> and http.response.code == 200
```

Comparer avec l'ensemble des chemins scannés (nombreux 404) pour identifier lequel retourne un code de succès (200/301/302), signalant l'existence réelle du répertoire côté serveur.

#### 8. Identifiants de connexion utilisés

Filtre :
```
http.request.method == "POST" and http.request.uri == "<CHEMIN_LOGIN>"
```

- Clic droit sur un paquet du résultat → **Follow → HTTP Stream**.
- Wireshark reconstruit l'échange TCP complet en clair (requête + réponse).
- Le corps du POST (paramètres `username=`/`password=` ou équivalent) est visible directement en texte lisible.
- Naviguer entre les tentatives successives via le bouton **Next stream** en bas de la fenêtre Follow Stream.
- Identifier la tentative dont la réponse HTTP diffère des autres (code de redirection au lieu d'un réaffichage du formulaire) : c'est la tentative réussie.

#### 9. Script malveillant uploadé

Filtre :
```
http.request.method == "POST" and http.request.uri == "<CHEMIN_ADMIN_PANEL>"
```

- **Follow → HTTP Stream** sur le paquet identifié.
- Le corps multipart/form-data est visible en clair, incluant le nom du fichier uploadé (`filename="..."`) et son contenu.

#### Astuce transverse : colonnes personnalisées

Pour éviter de rouvrir Follow Stream à répétition, ajouter des colonnes personnalisées via clic droit sur les champs suivants dans le panneau de détails du paquet → **Apply as Column** :

- `http.request.uri`
- `http.request.method`
- `http.host`
- `http.response.code`

Cela transforme la liste de paquets en vue tabulaire navigable, équivalente à une extraction en ligne de commande mais interactive.

#### Notes méthodologiques générales

- Toujours travailler en filtrant d'abord sur l'IP source de l'attaquant une fois identifiée, pour réduire le bruit du trafic légitime.
- Le tri chronologique (ordre naturel des paquets) est essentiel pour reconstituer la timeline d'attaque : reconnaissance → exploitation → post-exploitation.
- Follow HTTP Stream est l'outil central pour lire le contenu applicatif complet (requête + réponse) sans décodage manuel, tant que le trafic est en clair (HTTP, pas HTTPS).
- Pour du texte URL-encodé dans une URI, Wireshark n'effectue pas toujours le décodage automatique dans la liste de paquets : un outil externe de décodage reste nécessaire pour lecture finale.
