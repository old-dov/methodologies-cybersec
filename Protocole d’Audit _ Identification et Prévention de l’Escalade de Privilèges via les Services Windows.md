### Protocole d’Audit : Identification et Prévention de l’Escalade de Privilèges via les Services Windows

#### 1\. Fondamentaux de la Sécurité des Objets Windows

Dans l'architecture Windows, la sécurité est structurée par une séparation rigoureuse entre le  **User Mode (Ring 3\)**  et le  **Kernel Mode (Ring 0\)** . Cette frontière de confiance est maintenue par le noyau, qui interdit aux applications utilisateur l'accès direct aux ressources matérielles. Pour un auditeur, l'enjeu consiste à identifier les failles permettant une transition illégitime vers le contexte système. Cette dynamique repose sur deux composants pivots : l' **Object Manager**  et le  **Security Reference Monitor (SRM)** .L'Object Manager est le gestionnaire central de toutes les ressources (fichiers, processus, clés de registre, services). Lorsqu'un processus tente d'accéder à un objet via son chemin utilisateur (ex: C:\\), l'Object Manager résout ce chemin en un chemin natif du noyau (ex: \\Device\\HarddiskVolume1). C'est à ce stade qu'intervient le SRM, l'unique arbitre de sécurité, qui compare le jeton d'accès (Access Token) de l'utilisateur avec le  **Security Descriptor**  de l'objet pour valider les droits.Chaque objet possède un Security Descriptor contenant :

* **DACL (Discretionary Access Control List) :**  Liste d'entrées (ACE) définissant explicitement qui peut accéder à l'objet et avec quelles permissions (Lecture, Écriture, Modification).  
* **SACL (System Access Control List) :**  Liste dédiée à l'audit, déterminant quelles actions doivent générer un événement dans les journaux de sécurité.La compromission de la segmentation Ring 3/Ring 0 survient quasi systématiquement lorsqu'une mauvaise configuration de la DACL permet à un utilisateur non privilégié de manipuler un objet géré par le  **Service Control Manager (SCM)** , lequel s'exécute en Ring 0\.

#### 2\. Méthodologie d'Identification des Services Vulnérables

Le vecteur d'attaque  **Service Binary Hijacking**  cible les services s'exécutant sous le compte NT AUTHORITY\\SYSTEM. Si la DACL d'un service ou de son répertoire d'installation est trop permissive, un attaquant peut remplacer le binaire légitime par un code malveillant.

##### Procédure d'Inspection technique

L'outil icacls est impératif pour auditer les permissions. L'auditeur ne doit pas se limiter au répertoire racine, mais inspecter l'arborescence complète et le binaire lui-même à l'aide du flag /t (récursivité).**Tableau de référence des permissions symboliques :**| Symbole | Permission | Impact en Audit de Sécurité || \------ | \------ | \------ || **F** | Contrôle total | **Critique :**  Propriété totale de l'objet. || **M** | Modification | **Critique :**  Permet de supprimer ou remplacer le binaire. || **RX** | Lecture et exécution | Standard de sécurité recommandé pour le groupe Users. || **W** | Écriture | **Risque élevé :**  Permet l'injection de fichiers ou de DLL. || **(I)** | Héritée | **Indicateur :**  La permission provient d'un dossier parent (ex: C:\\). |  
**Commande opérationnelle d'audit :**  
icacls "C:\\Program Files\\NomDuService" /t

L'identification du droit  **Modification (M)**  associé au flag  **(I)**  révèle souvent une vulnérabilité structurelle : l'application a été installée dans un répertoire héritant de permissions laxistes du lecteur racine, permettant à n'importe quel utilisateur standard de détourner l'exécution système.

#### 3\. Étude de Cas : Vulnérabilité du Service 'PrintHelperSvc'

L'audit du service PrintHelperSvc sur la machine WIN11 illustre une défaillance critique de configuration DACL due à une mauvaise isolation du chemin d'installation.

##### Faits Techniques

* **Chemin d'installation :**  C:\\Program Files\\PrintHelper  
* **Preuve Technique (icacls) :**  BUILTIN\\Users:(I)(M)  
* **Diagnostic :**  Le flag  **(I)**  indique que les droits de  **Modification (M)**  sont hérités. Dans ce cas précis, le service a été déployé sans rompre l'héritage des permissions parentes ou sans restreindre les droits du groupe Users.Cette configuration permet à un utilisateur local d'écraser PrintHelperSvc.exe. Étant donné que ce service est orchestré par le SCM pour s'exécuter avec les privilèges SYSTEM, le remplacement du binaire garantit une escalade de privilèges totale dès que le service est sollicité ou redémarré.

#### 4\. Analyse du Vecteur d'Exploitation (Kill Chain)

L'auditeur doit maîtriser la chaîne d'attaque pour valider l'exploitabilité réelle. Dans un environnement durci, l'attaquant doit souvent contourner des restrictions réseau et des solutions de sécurité.

##### Séquence Opérationnelle (Basée sur le rapport GalacticGazers)

1. **Préparation du Payload :**  Compilation d'un binaire helper.exe (Reverse Shell) via x86\_64-w64-mingw32-gcc utilisant la bibliothèque winsock2.  
2. **Injection via Base64 :**  En raison de restrictions réseau (blocage HTTP), le binaire est converti en chaîne Base64 et reconstruit manuellement sur la cible via le terminal pour éviter les flux de téléchargement suspects.  
3. **Neutralisation des Défenses :**  Dans le cadre de cet audit (avec les identifiants administratifs de test fournis), Microsoft Defender a été désactivé pour permettre l'exécution du payload :  
4. **Hijacking et Exécution :**  L'arrêt du service est nécessaire pour libérer le binaire original.  *Note technique :*  L'arrêt d'un service requiert généralement des droits spécifiques (ex: Remote Management Users) ou un redémarrage système si l'utilisateur n'a pas les privilèges Stop-Service.**Logique d'exploitation (PowerShell) :**

Stop-Service PrintHelperSvc  
\# Remplacement par le payload injecté en Base64 et décodé  
Copy-Item "C:\\Temp\\helper.exe" "C:\\Program Files\\PrintHelper\\PrintHelperSvc.exe" \-Force  
Start-Service PrintHelperSvc

Le SCM charge alors le binaire malveillant en Ring 0, octroyant un accès NT AUTHORITY\\SYSTEM.

#### 5\. Stratégies de Remédiation et Durcissement (Hardening)

La sécurisation des services ne doit pas être réactive, mais structurelle, basée sur le principe du moindre privilège.

##### Directives Impératives de Remédiation

* **Correction Mandataire des DACL :**  Le groupe Users (SID S-1-5-21-...) doit être strictement limité aux droits de  **Lecture et Exécution (RX)** . Toute permission de Modification (M) ou d'Écriture (W) sur un binaire de service doit être considérée comme une vulnérabilité critique.  
* **Isolation de l'Héritage :**  Lors de l'installation de services tiers, l'héritage des permissions depuis la racine du disque doit être désactivé. La propriété (Ownership) des dossiers sensibles doit être exclusivement assignée à TrustedInstaller ou au groupe Administrators.  
* **Surveillance Active (SACL) :**  L'activation de l'audit de l'accès aux objets est cruciale pour détecter le remplacement de binaires.**Configuration de l'audit système (nécessite une invite élevée) :**

\# Activation de l'audit du File System  
auditpol /set /subcategory:"File System" /success:enable /failure:enable

\# Vérification de la politique d'accès aux objets  
auditpol /get /category:"Object Access"

Le durcissement des services Windows est un processus continu. Un audit rigoureux via icacls couplé à une surveillance étroite des modifications de fichiers via les SACL constitue la défense la plus robuste contre l'escalade de privilèges.  
