# Manuel de Procédures Forensiques : Acquisition et Analyse Multi-Vecteurs

#### 1\. Fondements et Cadre Méthodologique de l'Investigation

Dans le cadre d'une réponse aux incidents (IR), la rigueur méthodologique est le rempart indispensable contre l'irrecevabilité des preuves. L'objectif d'un expert DFIR est de transformer des données volatiles et persistantes en évidences juridiquement exploitables, tout en garantissant une chaîne de possession ( *Chain of Custody* ) sans faille. Toute manipulation technique doit être documentée et reproductible selon les standards internationaux.

##### Le Cycle de Vie Forensique (NIST SP 800-86)

La structuration de l'enquête repose sur les quatre phases du NIST, imposant des impératifs techniques stricts :

1. **Collection :**  Acquisition des données via des bloqueurs d'écriture ( *Write blockers* ). L'intégrité est scellée par un hachage cryptographique systématique ( **SHA-256** ).  
2. **Examination :**  Filtrage des données, reconstruction des systèmes de fichiers et extraction des métadonnées.  
3. **Analysis :**  Reconstitution de la  *timeline*  des événements et identification des TTP (Tactiques, Techniques, Procédures) de l'attaquant.  
4. **Reporting :**  Production d'un rapport synthétisant les faits, les limites techniques et les préconisations.

##### Standards de Validité Légale

Pour satisfaire aux exigences judiciaires, l'investigateur doit respecter la norme  **ISO/IEC 27041**  (validation des outils) et les cinq critères du  **Standard de Daubert**  :

* **Testabilité :**  La méthode a-t-elle été testée ?  
* **Peer Review :**  A-t-elle été révisée par des pairs ?  
* **Taux d'erreur connu :**  Les limites de précision sont-elles documentées ?  
* **Standards et contrôles :**  Existence de protocoles de contrôle qualité.  
* **Acceptation générale :**  La méthode est-elle reconnue par la communauté scientifique ?

##### Principes de Volatilité

L'ordre d'acquisition est dicté par la vitesse de disparition des données. La RAM, contenant les preuves "vivantes" (processus actifs, clés de chiffrement, connexions C2), doit impérativement être capturée avant toute intervention sur le stockage physique. Éteindre une machine sans capture préalable condamne définitivement l'accès aux menaces "fileless".*La préservation de l'état volatil effectuée, l'investigateur procède à la gestion des supports de stockage physiques pour l'analyse historique.*

#### 2\. Protocole d'Acquisition et d'Analyse du Stockage (Disk Forensics)

Le passage du HDD au SSD a radicalement modifié la donne forensique. Si le HDD permettait une récupération aisée dans le  *unallocated space* , les technologies SSD comme le  **TRIM**  et le  **Wear Leveling**  agissent comme des dispositifs "anti-forensics by design", purgeant physiquement les données supprimées pour optimiser les performances, rendant la récupération souvent illusoire.

##### Stratégies d'Acquisition : FTK Imager vs KAPE

Le choix de l'outil dépend du temps imparti et de la profondeur d'analyse requise.| Caractéristique | FTK Imager (Acquisition complète) | KAPE (Acquisition ciblée / Triage) || \------ | \------ | \------ || **Objectif** | Image bit-à-bit exhaustive (.E01 / .dd). | Collecte chirurgicale d'artefacts. || **Vitesse** | Lente (dépend du volume physique). | Très rapide (quelques minutes). || **Intégrité des Artefacts** | Copie brute incluant le  *slack space* . | Utilise  **EZParser (EvtxECmd, MFTECmd)** . || **Modes d'acquisition** | Physical, Logical, VSS (Snapshots). | Targets (Collecte) et Modules (Traitement). |

##### Méthodologie d'Analyse sous Autopsy

Le  *workflow*  opérationnel débute par l'ingestion de l'image disque ou des fichiers collectés par KAPE. Les "Ingest Modules" critiques à activer sont :

* **Recent Activity & Web Artifacts :**  Extraction de la navigation et des documents récents.  
* **Hash Lookup :**  Identification de malwares via des bases de signatures (ex: NSRL).  
* **Keyword Search :**  Recherche de chaînes spécifiques (ex: "Mimikatz").  
* **EXIF Parser :**  Analyse des métadonnées d'images pour la localisation ou l'identification d'outils.

##### Exploration des Artefacts par OS

* **Windows :**  Focalisation sur la  **MFT**  (Master File Table), les registres (SYSTEM, SOFTWARE), le  **Prefetch**  (historique d'exécution), les  **Shellbags**  et le  **$LogFile**  (journal NTFS permettant de détecter les tentatives d'anti-forensics).  
* *Point de vigilance :*  La valeur ProductName en registre peut indiquer "Windows 10 Pro" alors que le système est un Windows 11\. Seul le numéro de  **Build (ex: 26100 pour 24H2)**  fait foi.  
* **Linux :**  Utilisation de  **dcfldd**  pour l'imagerie. Analyse des logs dans /var/log (auth.log, syslog), de l'historique \~/.bash\_history et des persistances via les  *Cron jobs* .*L'image disque fige les traces persistantes, mais seule l'analyse mémoire révèle les vecteurs d'attaque en cours d'exécution.*

#### 3\. Analyse Avancée de la Mémoire Vive (Memory Forensics)

La mémoire vive agit comme une "caméra de surveillance" révélant les injections de code et les connexions réseau actives que le disque ignore.

##### Acquisition avec WinPMEM

Pour minimiser l'empreinte en RAM (principe de moindre impact), utilisez l'exécutable signé  **WinPMEM**  via PowerShell (Admin) :  
.\\go-winpmem\_amd64\_1.0-rc1.exe acquire C:\\Users\\Public\\memory.raw

Un hachage immédiat du dump memory.raw est obligatoire pour garantir l'intégrité.

##### Exploitation via Volatility 3

Volatility 3 nécessite des symboles PDB (gérés via un serveur de symboles) pour traduire les structures internes de l'OS.  **Attention :**  Le plugin windows.netscan peut être instable sur certaines versions récentes de Windows 11\.**Catalogue opérationnel :**

* windows.info : Identification du build exact (Crucial pour distinguer Win 10/11).  
* windows.pslist / pstree : Hiérarchie des processus.  
* windows.malfind : Détection de régions mémoire suspectes (PAGE\_EXECUTE\_READWRITE) sans fichier associé sur disque.  
* windows.cmdline : Extraction des arguments de lancement (ex: scripts PowerShell encodés).

##### Cas Pratique : Incident SkyLink Designs (Dump SKYLNK\_WS\_404.raw)

L'analyse révèle que le système est un  **Windows 11 (Build 26100\)** , malgré un registre affichant fallacieusement "Windows 10".

* **Vecteur :**  L'employée a exécuté un script cleaning.ps1 (PID 10452\) avec l'argument \-ExecutionPolicy Bypass.  
* **Chaîne d'exécution :**  Windows Terminal (PID 5400\) \-\> PowerShell (PID 2608\) \-\> cleaning.ps1 (PID 10452).  
* **Preuve d'intrusion :**  Le processus PID 6480 affichait le flag JEDHA{403103445533112} en clair dans sa ligne de commande.

##### Notion de Page Tables

Volatility utilise les  **Page Tables**  pour la translation d'adresses : il convertit les  **adresses virtuelles**  (vue isolée propre à chaque processus) en  **adresses physiques**  (RAM réelle). Cette étape est le fondement technique permettant d'isoler le contenu d'un processus malveillant du reste du système.*L'identification de processus suspects mène inévitablement à l'analyse de leurs communications externes.*

#### 4\. Investigation Réseau et Détection d'Exfiltration

La forensic réseau trace les mouvements latéraux et confirme l'exfiltration de données, souvent via des protocoles détournés.

##### Maîtrise de Wireshark

L'analyse commence par  **Statistics \-\> Protocol Hierarchy** . Un volume anormal de trafic DNS (ex: \>90% des paquets) est un indicateur fort de tunneling.  **Filtres critiques :**

* tcp.flags.syn \== 1 && tcp.flags.ack \== 0 : Identification de scans de ports.  
* http.response.code \== 200 : Extraction des accès réussis (isole le signal des erreurs 404 de brute force).

##### Focus : Détection du Tunneling DNS (Cas BrewByte)

L'attaquant fragmente les données sensibles et les encode dans des sous-domaines vers un serveur C2 (ex: oast.me).

1. **Identifier le domaine :**  Recherche de domaines fréquents comme \*.oast.me.  
2. **Décoder le HEX :**  Les sous-domaines (ex: 2d2d2d...) sont extraits et convertis.  
3. **Reconstruction via tshark :**  
4. **Résultat :**  L'exfiltration de  **519 tranches de données**  a permis de reconstituer une  **clé privée OpenSSH**  appartenant à student@97a0efa3a2db.

##### Analyse du Trafic Chiffré

Pour analyser le trafic HTTPS/TLS, l'importation des clés est nécessaire via  **Edit \> Preferences \> Protocols \> TLS** . Sans ces secrets, le contenu demeure un amas de données inexploitable ("gibberish").

#### 5\. Synthèse, Qualification MITRE ATT\&CK et Remédiation

##### Cartographie MITRE ATT\&CK

L'incident SkyLink et l'exfiltration DNS BrewByte se traduisent par la matrice suivante :| Technique | ID | Preuve Technique || \------ | \------ | \------ || **PowerShell** | T1059.001 | Exécution de cleaning.ps1 (PID 10452). || **Impair Defenses** | T1562 | Utilisation de \-ExecutionPolicy Bypass. || **User Execution** | T1204 | Téléchargement et exécution manuelle d'un script suspect. || **DNS Protocol** | T1071.004 | Tunneling via le domaine oast.me. || **Exfiltration over Alt Protocol** | T1048 | Vol d'une clé SSH en 519 chunks via requêtes DNS. |

##### Directives de Reporting

Tout rapport doit inclure :

* **Hashes (SHA-256)**  des preuves pour garantir l'intégrité.  
* **Timeline enrichie :**  Corrélation entre l'exécution du script (RAM) et l'exfiltration (Réseau).  
* **Inventaire des artefacts :**  PIDs, adresses IP de C2, domaines et fichiers créés.

##### Recommandations Post-Incident

1. **Contrôle d'Exécution :**  Imposer une politique PowerShell AllSigned et restreindre les privilèges utilisateurs.  
2. **Visibilité :**  Activer le  **ScriptBlock Logging**  pour tracer le contenu réel des commandes PowerShell.  
3. **Réponse Immédiate :**  Révocation de la clé SSH compromise et rotation de tous les secrets associés à l'identité student.  
4. **Forensic Linux :**  Sur les serveurs compromis, déployer  **LiME**  pour la capture RAM et  **dcfldd**  pour l'imagerie disque.*La validité des conclusions repose sur la reproductibilité stricte de ces méthodes, conformément aux standards ISO.*

