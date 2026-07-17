# Protocole d'Investigation Numérique : Détection d'Intrusion et Surveillance des Processus Critiques

#### 1\. Fondamentaux de l'Architecture de Sécurité Windows

Pour un analyste SOC, la maîtrise des mécanismes internes de Windows est une nécessité stratégique pour transformer des flux de données brutes en renseignements exploitables. La visibilité granulaire constitue le socle de la défense : sans une compréhension rigoureuse des interactions entre les jetons d'accès (Tokens), le sous-système LSASS et le moniteur SRM, un défenseur est incapable de détecter les mouvements latéraux et les escalades de privilèges subtiles.

##### Fiches Techniques des Composants Stratégiques

* **Access Tokens (Jetons d'accès) :**  
* **Rôle :**  Structure de données encapsulant l'identité de sécurité d'un processus (SID, groupes, privilèges).  
* **Niveaux d'Intégrité (Integrity Levels) :**  Crucial pour la segmentation. Les niveaux (Low, Medium, High, System) déterminent les capacités d'interaction. Un jeton "High" est impératif pour les actions administratives (UAC), même si l'utilisateur possède les droits requis.  
* **Impact ("So What?") :**  La compromission ou la manipulation d'un jeton permet l'usurpation d'identité ou l'élévation de droits sans modification de compte.  
* **LSASS (Local Security Authority Subsystem Service) :**  
* **Rôle :**  Service tournant en mode utilisateur sous SYSTEM, responsable de l'authentification et de la génération des tokens via l'appel NtCreateToken.  
* **Impact ("So What?") :**  Cible prioritaire car il contient le matériel d'authentification en mémoire. Sa compromission est le point de passage obligé pour l'extraction de secrets.  
* **SRM (Security Reference Monitor) :**  
* **Rôle :**  Composant critique du  **noyau (Kernel)** . Il arbitre chaque accès à un objet en comparant le token du processus au DACL de l'objet.  
* **Impact ("So What?") :**  Arbitre final du système. Toute faille dans la logique des ACL ou la structure des jetons permet de contourner le SRM pour réaliser une escalade de privilèges.

##### Comparatif Technique : Primary vs. Impersonation Tokens

Type de Jeton,Description Technique,Vecteur d'Exploitation Type  
Primary Token,Définit les capacités de base d'un processus au lancement (ex: powershell.exe).,Établissement de la base de privilèges d'une session.  
Impersonation Token,Permet à un service d'agir temporairement sous l'identité d'un client.,"Exploitation de  Named Pipes  (ex: lsass, ntsvcs) où un service malveillant usurpe l'identité d'un client privilégié."  
La maîtrise de ces composants est indispensable pour interpréter les logs : chaque événement est la trace d'une interaction entre un jeton, le LSASS et le SRM.

#### 2\. Cadre de Surveillance : Exploitation des Journaux d'Événements

Priorisez la corrélation d'événements à l'analyse isolée. Le bruit de fond système masque les signaux faibles ; seule la séquence chronologique permet de lever le doute sur une intention malveillante.

##### Catégorisation des Event IDs Critiques

* **Logon & Authentification :**  4624 (Succès), 4625 (Échec), 4648 (Identifiants explicites), 4776 (NTLM).  
* **Privilèges & Jetons :**  4672 (Privilèges spéciaux), 4696 (Assignation de token), 4719 (Politique d'audit modifiée).  
* **Processus & Services :**  4688 (Création), 4689 (Terminaison), 4697 (Création de service).  
* **Accès aux Objets :**  4656/4663 (Tentative d'accès), 4660 (Suppression).  
* **Système & Defender :**  1102 (Journal effacé), 1116/1117 (Détection de malware).

##### Analyse de la Séquence d'Attaque (4624 → 4672 → 4688 → 4697 → 1102\)

1. **4624 / 4672 :**  Accès initial suivi de l'obtention de privilèges administratifs.  
2. **4688 :**  Création de processus.  **Attention :**  Cet événement n'est exploitable que si l'audit de la  **ligne de commande**  est activé via GPO, faute de quoi l'analyste ignore l'action réelle (ex: powershell.exe \-enc...).  
3. **4697 :**  Installation d'un service pour garantir la persistance.  
4. **1102 :**  Effacement du journal de sécurité. Il s'agit d'une  **tentative active de dissimulation**  post-compromission.

##### Protocole d'Activation de l'Audit Avancé

Exécutez ces commandes pour garantir une visibilité exhaustive, incluant les mouvements latéraux et les manipulations de privilèges :  
auditpol /set /category:"Detailed Tracking" /subcategory:"Process Creation" /success:enable  
auditpol /set /category:"Detailed Tracking" /subcategory:"RPC Events" /success:enable  
auditpol /set /category:"Privilege Use" /subcategory:"Token Right Adjusted" /success:enable

#### 3\. Investigation Dynamique avec la Suite Sysinternals

##### Analyse Comportementale via Process Monitor (Procmon)

Procmon est l'outil de référence pour l'analyse en temps réel. Pour isoler l'activité suspecte, filtrez impérativement le bruit de fond des processus légitimes (ex: exclure MsMpEng.exe).

* **Intégrité des fichiers :**  Utilisez handle.exe pour identifier les processus verrouillant des fichiers sensibles (ex: bases de données de mots de passe ou ruches de registre).  
* **Détection d'injection et DLL Hijacking :**  Surveillez les événements  **"Load Image"**  dans Procmon. Le chargement de DLL non signées ou provenant de répertoires utilisateur par des processus critiques comme svchost.exe est un indicateur fort d'injection de code.

#### 4\. Analyse des Tentatives d'Escalade de Privilèges et Accès au SAM

Le SAM (Security Account Manager) et LSASS sont les cibles prioritaires pour l'extraction de secrets locaux.

##### Mécanismes d'Extraction et "LOLBins"

* **Extraction Online :**  Les attaquants privilégient désormais l'usage de commandes natives comme reg save HKLM\\SAM et reg save HKLM\\SYSTEM. Ces  **LOLBins**  (Living Off the Land Binaries) sont moins susceptibles de déclencher une alerte EDR que Mimikatz.  
* **Nécessité du fichier SYSTEM :**  L'extraction du SAM seul est inutile ; le fichier SYSTEM contient la  **SysKey**  indispensable au déchiffrement des hashs NTLM.  
* **Privilèges Requis :**  L'accès live nécessite SYSTEM ou l'activation de SeDebugPrivilege.

##### Checklist d'Investigation : Dump de Mémoire et Registre

*  Présence de fichiers .save, .dump ou .hiv inhabituels.  
*  Exécution de procdump.exe ciblant lsass.exe.  
*  Événement 4672 confirmant l'usage de SeDebugPrivilege.  
*  Existence de  **Named Pipes**  suspects (ex: pipes créés par des processus non-système pour l'impersonation).

#### 5\. Protocole de Vérification de l'Intégrité de Winlogon et Persistance

Winlogon est un processus critique gérant les sessions interactives sous SYSTEM. Sa compromission assure une persistance totale.

##### Points de Défense et de Rupture

* **Secure Attention Sequence (SAS) :**  Winlogon gère exclusivement le Ctrl+Alt+Del. Cette séquence matérielle est la protection fondamentale contre les  **"Fake Logon Screens"**  (écrans de connexion factices).  
* **Héritage de Token :**  Le flux standard (Winlogon → LSASS → userinit.exe → explorer.exe) peut être détourné. Un malware substituant userinit.exe héritera automatiquement du jeton de sécurité de la session.

##### Analyse des Clés de Registre de Persistance (Chemins Complets)

Vérifiez l'intégrité des valeurs dans :

* HKLM\\Software\\Microsoft\\Windows NT\\CurrentVersion\\Winlogon\\Shell (Défaut : explorer.exe)  
* HKLM\\Software\\Microsoft\\Windows NT\\CurrentVersion\\Winlogon\\Userinit (Défaut : C:\\Windows\\system32\\userinit.exe,)**Mise en garde :**  Toute interruption forcée du processus Winlogon provoque un  **BSOD (Blue Screen of Death)**  immédiat.

#### 6\. Synthèse des Procédures de Remédiation et Durcissement

##### Méthodologie Opérationnelle

Partir de la détection historique par les  **Event IDs** , puis valider les comportements en temps réel via les outils  **Sysinternals** .

##### Recommandations de Durcissement

1. **LSA Protection & Credential Guard :**  Activez ces isolations pour rendre la mémoire de LSASS inaccessible aux outils de dump traditionnels.  
2. **BitLocker :**  Déployez le chiffrement pour interdire l'analyse forensique  **offline**  (ex: montage du disque via un OS tiers comme Linux pour extraire le SAM).  
3. **Principe de Moindre Privilège :**  Auditez et restreignez drastiquement l'assignation de SeDebugPrivilege.  
4. **Mise à jour de l'Audit :**  La visibilité doit évoluer avec les techniques d'évasion ; la surveillance des lignes de commande et des événements RPC est désormais le standard minimal pour tout analyste DFIR.

