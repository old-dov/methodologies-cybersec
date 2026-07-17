# Fiche de Méthode : Le Flux de Travail Metasploit (du Scan à la Session)

Cette fiche méthodologique détaille la démarche rigoureuse de "Red Teaming" nécessaire pour mener une intrusion avec le framework Metasploit, en transformant une simple reconnaissance réseau en une session de contrôle avancée.

#### 1\. Fondamentaux : Le Triptyque de l'Exploitation

La maîtrise de Metasploit exige une distinction nette entre les trois piliers de l'attaque. Ne confondez jamais l'accès (la vulnérabilité) avec l'outil (l'exploit) ou l'action (le payload).| Concept | Définition | Rôle dans l'attaque | Exemple concret || \------ | \------ | \------ | \------ || **Vulnérabilité** | Faiblesse logique, de code ou de configuration. | Constitue la "porte d'entrée" potentielle. | **Buffer Overflow** , SQL Injection, Identifiants par défaut. || **Exploit** | Code conçu pour tirer parti d'une vulnérabilité. | Déclenche un comportement non prévu pour forcer l'accès. | Script exploitant  **EternalBlue**  (CVE-2017-0144). || **Payload** | Code exécuté sur la cible après l'exploitation. | Définit l'objectif final du pentester. | **Reverse Shell** , Meterpreter, Script d'exfiltration. |  
**Transition :**  La compréhension de ces briques est le préalable indispensable à toute action sur Metasploit.

#### 2\. Phase Préparatoire : Connaissance de l'Environnement

Avant d'initialiser la  **msfconsole** , exigez une visibilité totale sur votre topologie réseau. Une erreur de configuration ici garantit l'échec de tout  *reverse shell* .

* **Inspection impérative des interfaces :**  Utilisez la commande  **ip a**  pour identifier l'adresse IP exacte de votre machine d'attaque.  
* **Vérification de la joignabilité :**  
* Si vous opérez via un VPN, votre  **LHOST**  doit impérativement être l'adresse de l'interface  **tun0** .  
* Dans un environnement de conteneurs ou de lab local, utilisez l'adresse de l'interface bridge (souvent  **eth0** ).  
* **Règle d'or :**  L'adresse de rappel ( **LHOST** ) doit être  *atteignable*  par la cible. Une IP physique est inutile si la cible est isolée sur un réseau virtuel.**Transition :**  Une fois l'environnement configuré, le pentester peut passer à la phase de collecte d'informations.

#### 3\. Phase 1 — Reconnaissance et Identification du Vecteur

Utilisez  **nmap**  non pas comme un simple scanner, mais comme un outil de diagnostic de surface d'attaque.

##### Principes de scan (Méthode en double passe)

1. **Passe de découverte :**  Identifiez rapidement tous les ports ouverts ( **\-p-** ).  
2. **Passe de caractérisation :**  Exécutez une détection de version précise avec l'option  **\-sV** .  **C'est cette étape qui constitue le catalyseur de la phase suivante** , car elle permet d'orienter la recherche d'exploits vers des versions logicielles spécifiques.  
3. **Désactivation ICMP :**  Si l'hôte est protégé par un pare-feu, forcez le scan avec  **\-Pn** .

##### Identification du vecteur (Analyse critique)

Ne vous précipitez pas sur le premier port ouvert. Analysez :

* La présence d' **interfaces d'administration**  (ex: Tomcat Manager, interfaces Web).  
* L'utilisation de  **composants tiers**  (plugins, frameworks) souvent moins sécurisés que le noyau du système.  
* Les  **configurations par défaut**  : Testez systématiquement les couples identifiants/mots de passe classiques si une interface d'authentification est détectée.**Transition :**  Les données récoltées servent de base pour interroger la base de données de Metasploit.

#### 4\. Phase 2 — Recherche et Sélection de la Vulnérabilité

Face à une bibliothèque de plus de 2200 exploits, la sélection doit être méthodique. Utilisez  **search**  pour lister les modules et  **info**  pour en disséquer le fonctionnement.

##### Critères de sélection d'un module

* **Cohérence temporelle :**  Assurez-vous que la date de divulgation de la faille est compatible avec l'ancienneté du service cible.  
* **Mécanisme de l'exploit :**  Le module doit correspondre précisément au vecteur identifié (ex: n'utilisez pas un module d'upload si la faille est une injection SQL).  
* **Fiabilité (Rank) :**  Privilégiez les modules notés  **Excellent**  ou  **Great**  pour garantir la stabilité du service cible et éviter les plantages (DoS).**Transition :**  Une fois le module sélectionné avec  **use** , il est temps de configurer les paramètres d'attaque.

#### 5\. Phase 3 — Configuration Méthodique de l'Exploitation

La configuration est l'étape où la précision technique prévaut sur l'automatisme.

##### Guide des options indispensables

Option,Utilité pour le pentester  
RHOSTS,Adresse IP (ou plage) de la cible.  
RPORT,Port distant du service vulnérable.  
LHOST,Votre IP de rappel (interface accessible par la cible).  
LPORT,Port d'écoute sur votre machine (souvent 4444 par défaut).  
TARGET,"Spécifie l'architecture cible (OS, version)."  
**Note critique sur l'option**  **TARGET**  **:**  Le jeu de payloads compatibles affichés par Metasploit dépend strictement du  **TARGET**  choisi. Si votre payload favori n'apparaît pas, vérifiez la cohérence de votre cible.

##### Choix du Payload

* **Payload "Commande" :**  Pour une action unique et discrète (ex: lecture de /etc/passwd).  
* **Payload "Session" (Meterpreter) :**  Pour un contrôle interactif total.**Action impérative :**  Exécutez toujours  **show options**  pour une validation finale avant de lancer la commande  **run**  ou  **exploit** .**Transition :**  Avec une configuration validée, l'exécution du module vise l'obtention d'un accès.

#### 6\. Phase 4 — Post-Exploitation : La Puissance de Meterpreter

Meterpreter est un environnement de post-exploitation dynamique capable d'injecter des scripts et de manipuler le système en mémoire.

##### Les 5 commandes critiques de "Looting"

1. **sysinfo**  : Identifier précisément l'OS et l'architecture pour préparer l'escalade de privilèges.  
2. **getsystem**  : Tenter d'élever les privilèges vers le niveau SYSTEM (Windows) ou Root.  
3. **screenshot**  : Capturer l'activité visuelle de l'utilisateur pour collecter des informations contextuelles.  
4. **download**  : Exfiltrer des fichiers sensibles vers votre machine d'attaque.  
5. **upload**  : Transférer des outils de post-exploitation supplémentaires sur la cible.

##### Stratégies d'exfiltration et diagnostic

En cas de conteneur minimal sans outils réseau, privilégiez l' **écriture et lecture web** .

* **Astuce de diagnostic Red Team :**  Si vous ignorez où le service sert ses fichiers (web root), utilisez une commande pour écrire des informations de contexte (ex: pwd; ls) dans plusieurs emplacements probables (/var/www/html, /tmp, /var/www/static) simultanément. Le répertoire qui répond à votre requête HTTP confirmera votre vecteur d'exfiltration.**Transition :**  Pour des besoins spécifiques, Metasploit permet également de sortir des sentiers battus avec des modules personnalisés.

#### 7\. Cas Particuliers : Modules FILEFORMAT et Customisation

##### Les modules FILEFORMAT

Ces modules génèrent un fichier piégé (PDF, document Office) qui doit être livré manuellement.

* **Vecteur :**  Social Engineering ou formulaire d'upload.  
* **Tactique de contournement :**  Si un filtre bloque votre extension, renommez le fichier (ex: .ps en .jpg). Les services analysent souvent le contenu réel (Magic Bytes) plutôt que l'extension pour traiter le fichier.

##### Architecture interne d'un module personnalisé (Ruby)

Plutôt que de copier un script, comprenez les 4 composants fonctionnels exigés par Metasploit :

1. **Le Constructeur (Initialize) :**  Définit les métadonnées (Nom, CVE, Description, Target).  
2. **La fonction Check :**  Vérifie si la cible est réellement vulnérable avant de tenter l'exploitation (évite le bruit inutile).  
3. **La fonction Send Request :**  Gère la communication HTTP/Réseau et l'envoi du buffer d'attaque.  
4. **La fonction Exploit :**  Génère le payload final, l'insère dans l'exploit et déclenche la livraison.Pour intégrer votre création : placez le fichier .rb dans votre répertoire local, exécutez  **reload\_all**  et testez la syntaxe.**Transition :**  Conclure le document par un rappel sur l'éthique et la rigueur méthodologique.

#### 8\. Synthèse : Les 5 Réflexes d'Or du Pentester

1. **Analyse de version (**  **\-sV**  **) :**  Ne lancez jamais un exploit sans avoir confirmé la version exacte du service cible.  
2. **Reachabilité de l'IP (**  **LHOST**  **) :**  Vérifiez toujours via  **ip a**  que votre adresse de rappel est joignable depuis le segment réseau de la cible.  
3. **Validation de configuration :**  Le réflexe  **show options**  est votre dernière barrière contre une erreur de frappe qui pourrait faire planter un service de production.  
4. **Adaptabilité de l'exfiltration :**  Si le canal interactif échoue, sachez rebondir sur une exfiltration via le répertoire web en utilisant l'astuce du diagnostic multi-chemins.  
5. **Documentation et Méthode :**  Un pentest réussi n'est pas une intrusion chanceuse, c'est une suite d'étapes validées, documentées et reproductibles pour le rapport final.

