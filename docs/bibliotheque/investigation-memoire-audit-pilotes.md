# Protocole d'Investigation Numérique : Analyse de la Mémoire et Audit des Pilotes

#### 1\. Fondements de l’Architecture Mémoire sous Windows

Dans le domaine du DFIR (Digital Forensics and Incident Response), la mémoire vive constitue la source de vérité ultime. Tout code, qu'il soit légitime ou malveillant, doit impérativement transiter par le processeur (CPU) pour être exécuté. L'architecture matérielle dicte ici la structure logicielle : les registres CPU, zones de stockage ultra-rapides, manipulent les données via des registres dédiés (notamment  **R1**  pour l'adressage et  **R2**  pour la valeur). La taille de ces registres (32 ou 64 bits) est l'élément fondamental qui détermine la version de l'OS et ses capacités d'adressage.Le système Windows orchestre une dualité entre la  **mémoire physique (RAM)**  — une succession de conteneurs de 1 octet identifiés par une adresse unique — et la  **mémoire virtuelle** , un espace d'adressage privé simulé pour chaque processus. Cette abstraction résout trois problématiques critiques :

* **Dépassement de RAM :**  Évite le crash système via le mécanisme de  *paging*  (utilisation du fichier pagefile.sys).  
* **Écrasement inter-processus :**  Garantit qu'un programme ne peut modifier l'espace d'un autre.  
* **Fragmentation :**  Linéarise l'espace d'adressage pour les applications malgré la fragmentation physique.**L'analyse du Memory Management Unit (MMU) :**  Le MMU, s'appuyant sur les tables de pages, assure la traduction des adresses virtuelles en adresses physiques. Pour l'enquêteur, cette isolation est une barrière : la réalité physique est fragmentée. L'enjeu forensique consiste à reconstruire cette continuité pour identifier les injections de code. Comprendre ce mapping est le seul moyen de percer les techniques d'évasion qui tentent de masquer la présence de données malveillantes en RAM.

#### 2\. Surveillance et Analyse de la Mémoire Vive (Live Analysis)

L'analyse "à chaud" est impérative pour capturer des indicateurs volatils avant qu'ils ne soient altérés par une réponse incidente maladroite ou un redémarrage. L'opérateur doit établir une base de référence (baseline) pour détecter les anomalies de consommation.L'usage de PowerShell permet un triage rapide. La commande suivante est le standard pour isoler les processus suspects : Get-Process | Sort-Object \-Descending WorkingSet | Select-Object \-First 10 Name, WorkingSet  *Note technique : Un processus comme*  *explorer.exe*  *présente généralement un WorkingSet aux alentours de 170 MB. Tout écart significatif sans justification applicative doit être investigué.*| Outil | Usage Stratégique | Artefacts de Pages Analysés || \------ | \------ | \------ || **Resource Monitor** | Monitoring temps réel | Flux CPU/RAM et handles de fichiers ouverts. || **RAMMap** | Analyse de structure | Pages  *private*  (données propres),  *mapped*  (fichiers projetés), et  *driver locked* . |  
**Analyse du différentiel WorkingSet vs Virtual Memory :**  Le WorkingSetMB représente la RAM physique réellement occupée, tandis que VirtualMemoryMB inclut les pages déportées sur disque. Un écart anormal peut trahir une technique d'évasion : le malware peut forcer son propre  *paging*  pour minimiser sa signature en RAM physique, se logeant dans le pagefile.sys pour échapper aux scanners mémoires superficiels.

#### 3\. Méthodologie de Dump et Forensic de la Mémoire (Post-Mortem)

Le protocole forensique impose de geler l'état du système à un instant T pour préserver les preuves volatiles telles que les clés de chiffrement, les connexions réseau actives et les payloads injectés.La séquence d'acquisition et d'analyse recommandée est la suivante :

1. **Génération du Dump :**  Extraction de l'espace d'adressage d'un processus suspect via ProcDump (format .dmp).  
2. **Extraction de chaînes :**  Utilisation de l'utilitaire Strings (recherche ASCII et Unicode).  
3. **Analyse de structure :**  Emploi de Volatility pour reconstruire les structures de données du noyau.**Le levier stratégique de l'analyse de chaînes :**  L'extraction de chaînes de caractères dans un dump mémoire est souvent plus fructueuse qu'une analyse de fichiers sur disque. Les malwares modernes utilisent l'offuscation et le chiffrement au repos ; cependant, pour s'exécuter, le code doit être déchiffré en mémoire. Le dump mémoire contient donc souvent le payload en clair, les adresses IP de commande et contrôle (C2), et les identifiants dérobés qui n'apparaissent jamais sur le stockage permanent.

#### 4\. Audit des Pilotes Système et Analyse de la Driver Stack

Les pilotes opérant en mode noyau ( **Ring 0** ) possèdent un accès total aux ressources. C'est l'emplacement privilégié pour les rootkits, car ils peuvent intercepter les appels système pour masquer leur propre existence.L'audit doit décomposer la pile d'entrée/sortie (I/O Stack) gérée par l'I/O Manager :

1. **Bus Driver :**  Gère la communication physique via le  **HAL (Hardware Abstraction Layer)** .  
2. **Function Driver :**  Implémente la logique spécifique du périphérique (ex: kbdhid.sys pour les claviers).  
3. **Filter Driver :**  Insère des fonctionnalités additionnelles. C'est ici que se logent les EDR, mais aussi les rootkits.**L'intégrité de la chaîne de confiance :**  Le  **PnP Manager**  utilise le  **Hardware ID**  pour associer un pilote via le registre et les fichiers .inf. Le  **Configuration Manager**  finalise cette installation dans la base de registre. L'enquêteur doit utiliser driverquery /si ou sigcheck pour valider les signatures numériques. Un pilote non signé ou un "Filter Driver" non identifié à une "altitude" suspecte dans la stack I/O est une preuve de compromission du noyau, capable de bypasser toutes les protections de l'OS.

#### 5\. Investigation des Mécanismes de Persistance : Étude de Cas

La persistance assure la survie du malware au redémarrage. L'incident "GalacticGazers" démontre la simplicité et l'efficacité des vecteurs basés sur le registre Windows.**Autopsie de l'incident :**  L'attaquant a exploité la clé HKEY\_LOCAL\_MACHINE\\Software\\Microsoft\\Windows\\CurrentVersion\\Run. En y injectant un exécutable renommé calc.exe (mimétisme de la calculatrice Windows) et placé dans C:\\Users\\Public\\, il garantit une exécution systématique.**Vecteur d'escalade "Write-to-Higher-Privilege" :**  L'analyse forensique des permissions via icacls a révélé que le groupe BUILTIN\\Users possédait le  **Full Control (F)**  sur ce répertoire. Techniquement, cela crée un risque critique d'escalade de privilèges : n'importe quel utilisateur standard peut remplacer ce fichier par un code malveillant. Lors de la connexion d'un administrateur, le système exécute le binaire malveillant avec les privilèges de ce dernier, brisant totalement le cloisonnement de sécurité de Windows.

#### 6\. Protocoles de Remédiation et Durcissement du Système

La phase finale consiste à éradiquer la menace et à transformer les vulnérabilités identifiées en mesures de durcissement (hardening).**Actions de remédiation technique :**

1. **Nettoyage :**  Suppression des entrées de registre malveillantes et du binaire C:\\Users\\Public\\calc.exe.  
2. **Restauration de l'intégrité :**  Restriction des permissions sur les dossiers publics. icacls "C:\\Users\\Public\\calc.exe" /remove "Users"  
3. **Surveillance proactive :**  Activation de l'audit système via auditpol pour historiser les accès au système de fichiers. auditpol /set /subcategory:"File System" /success:enable /failure:enable**Vers une défense en profondeur :**  Le passage d'une posture réactive à une défense proactive nécessite l'analyse constante des logs d'audit générés. Enfin, l'aspect technique doit être complété par le facteur humain : une  **campagne de sensibilisation**  est indispensable pour informer les utilisateurs sur les risques liés aux répertoires partagés et à l'exécution de binaires non vérifiés. Seule une approche intégrée — mêlant analyse granulaire de la mémoire, audit rigoureux des pilotes noyau et surveillance stricte des mécanismes de persistance — garantit la résilience du système face aux menaces persistantes.

