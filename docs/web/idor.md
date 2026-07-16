Rapport Méthodologique : Audit de Sécurité et Détection de Failles IDOR
1. Objectif de l'audit
L'objectif de cette méthodologie est d'identifier, de tester et de valider la présence de vulnérabilités de type IDOR (Insecure Direct Object References / Références directes non sécurisées à un objet) au sein d'une application web. Cette faille survient lorsqu'une application fournit un accès direct à des objets (fichiers, bases de données, comptes) en se basant sur une clé ou un identifiant fourni par l'utilisateur, sans effectuer de contrôle d'accès approprié.

2. Phase de Reconnaissance et de Cartographie
Avant de chercher à exploiter une faille, il convient de comprendre comment l'application gère et expose ses ressources.

Identification des fonctionnalités clés : Repérer toutes les zones de l'application qui génèrent, stockent ou affichent des données utilisateur (historiques de chat, factures, profils, documents téléchargés).

Analyse des flux réseau : Utiliser les outils de développement du navigateur (F12, onglet Réseau) ou un proxy d'interception (comme Burp Suite) pour observer les requêtes HTTP générées lors de l'accès à ces ressources.

Repérage des paramètres cibles : Identifier les paramètres dans les URL (par exemple, /fichiers/123.txt, ?id=1001) ou dans le corps des requêtes POST/JSON qui semblent faire référence à des identifiants d'objets en base de données ou sur le système de fichiers.

3. Phase d'Analyse de la Vulnérabilité (Le "Pattern Analysis")
Une fois les paramètres cibles identifiés, il faut analyser leur structure pour déterminer s'ils sont prévisibles.

Évaluation de la prévisibilité : * L'identifiant est-il incrémentiel/séquentiel (ex: 1, 2, 3 ou 1001, 1002) ? Si oui, la probabilité d'une faille IDOR est élevée car il est facile de deviner les identifiants des autres utilisateurs.

L'identifiant est-il un UUID/GUID ou un hash (ex: de305d54-75b4-431b...) ? Si oui, l'exploitation directe par conjecture ("guessing") est difficile, mais la faille conceptuelle peut toujours exister si le contrôle d'autorisation est absent.

Analyse du stockage : Déterminer si les fichiers sont stockés de manière statique sur le serveur ou générés dynamiquement à la volée. Un accès direct à un chemin de fichier statique (ex: /uploads/logs/chat_99.txt) est un indicateur fort d'IDOR.

4. Phase de Test et d'Exploitation (Preuve de Concept)
Pour confirmer la vulnérabilité, il faut tester si le serveur applique des règles d'autorisation strictes.

+------------------+         Requête (ex: id=1001)        +------------------+
|  Navigateur de   | -----------------------------------> |  Serveur Web et  |
| l'utilisateur A  | <----------------------------------- | Base de Données  |
+------------------+     Données de l'utilisateur B       +------------------+
                            (Absence de contrôle)
Création d'un référentiel de test (Optionnel mais recommandé) : Si possible, utiliser deux comptes d'utilisateurs distincts (Utilisateur A et Utilisateur B).

Manipulation de paramètres (Parameter Tampering) :

Se connecter avec le compte de l'Utilisateur A.

Accéder à une ressource propre à A (identifiant X).

Modifier manuellement l'identifiant X par un identifiant Y (potentiellement associé à l'Utilisateur B ou simplement un nombre inférieur/supérieur dans la séquence).

Observation du comportement du serveur :

Scénario Vulnérable (IDOR confirmée) : Le serveur renvoie la ressource de l'utilisateur B (ou un message système d'un autre utilisateur). Le code de statut HTTP est généralement 200 OK.

Scénario Sécurisé : Le serveur bloque l'accès. Il renvoie une erreur d'autorisation de type 403 Forbidden, 401 Unauthorized ou redirige vers une page d'erreur/connexion.

5. Analyse de l'Impact et Recherche d'Informations Sensibles
Si l'IDOR est confirmée, l'étape suivante consiste à évaluer la gravité des données exposées.

Extraction de données : Parcourir ou scripter la récupération des objets exposés pour y rechercher des informations critiques (mots de passe, jetons d'API, données personnelles / PII).

Recherche de privilèges : Vérifier si la modification des identifiants permet d'accéder à des fonctions administratives (ex: passer d'un ID utilisateur classique à un ID administrateur).

6. Recommandations de Remédiation (Mesures Défensives)
Pour corriger cette vulnérabilité, l'équipe de développement doit mettre en œuvre les mesures suivantes :

Contrôle d'accès basé sur l'autorisation (Access Control) : Ne jamais faire confiance aux paramètres fournis par le client. À chaque requête, le serveur doit vérifier si l'utilisateur actuellement authentifié (via sa session ou son jeton JWT) possède les droits nécessaires pour accéder à l'objet demandé.

Utilisation de références indirectes non prévisibles : Remplacer les identifiants séquentiels ou directs par des identifiants uniques complexes et non devinables (comme des UUID cryptographiquement sûrs).

Cartographie d'accès (Access Control List - ACL) : Utiliser une table de correspondance temporaire et propre à la session de l'utilisateur pour lier une clé éphémère (ex: clé_fichier_1) au véritable identifiant de la base de données.