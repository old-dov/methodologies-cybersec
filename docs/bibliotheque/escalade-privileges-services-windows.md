# Protocole d’Audit — Identification et Prévention de l’Escalade de Privilèges via les Services Windows

## 1. Fondamentaux de la sécurité des objets Windows

Dans l'architecture Windows, la sécurité repose sur une séparation stricte entre le **User Mode (Ring 3)** et le **Kernel Mode (Ring 0)**. Cette frontière de confiance est maintenue par le noyau, qui interdit aux applications utilisateur l'accès direct aux ressources matérielles.

Pour un auditeur, l'enjeu est d'identifier les failles permettant une transition illégitime vers le contexte système. Cette dynamique s'appuie sur deux composants pivots :

- **Object Manager** : gestionnaire central de toutes les ressources Windows, comme les fichiers, processus, clés de registre et services.
- **Security Reference Monitor (SRM)** : arbitre de sécurité chargé de comparer le jeton d'accès de l'utilisateur avec le descripteur de sécurité de l'objet ciblé.

Lorsqu'un processus tente d'accéder à un objet via un chemin utilisateur, par exemple `C:\`, l'Object Manager le résout en chemin natif du noyau, par exemple `\Device\HarddiskVolume1`. Le SRM décide ensuite si l'accès est autorisé.

Chaque objet possède un **Security Descriptor** contenant notamment :

- **DACL (Discretionary Access Control List)** : liste des permissions explicites accordées sur l'objet.
- **SACL (System Access Control List)** : liste dédiée à l'audit, indiquant quelles actions doivent générer des événements de sécurité.

La compromission de la segmentation Ring 3 / Ring 0 survient très souvent lorsqu'une mauvaise configuration de DACL permet à un utilisateur non privilégié de manipuler un objet géré par le **Service Control Manager (SCM)**, lequel orchestre l'exécution de services avec des privilèges élevés.

## 2. Méthodologie d'identification des services vulnérables

Le vecteur **Service Binary Hijacking** cible les services qui s'exécutent sous le compte `NT AUTHORITY\SYSTEM`. Si la DACL d'un service ou de son répertoire d'installation est trop permissive, un attaquant peut remplacer le binaire légitime par un exécutable malveillant.

### Procédure d'inspection technique

L'outil `icacls` est central pour auditer les permissions. L'auditeur ne doit pas se limiter au répertoire racine, mais inspecter l'arborescence complète ainsi que le binaire lui-même à l'aide du flag de récursivité.

| Symbole | Permission | Impact en audit de sécurité |
|---|---|---|
| **F** | Contrôle total | **Critique** : propriété complète de l'objet |
| **M** | Modification | **Critique** : permet de supprimer ou remplacer le binaire |
| **RX** | Lecture et exécution | Standard de sécurité recommandé pour le groupe Users |
| **W** | Écriture | **Risque élevé** : permet l'injection de fichiers ou de DLL |
| **(I)** | Héritée | **Indicateur** : la permission provient d'un dossier parent |

**Commande opérationnelle d'audit :**

```powershell
icacls "C:\Program Files\NomDuService" /t
```

L'identification du droit **Modification (M)** associé au flag **(I)** révèle souvent une vulnérabilité structurelle : l'application a été installée dans un répertoire qui hérite de permissions trop larges, permettant à un utilisateur standard de détourner une exécution système.

## 3. Étude de cas — vulnérabilité du service PrintHelperSvc

L'audit du service `PrintHelperSvc` sur la machine `WIN11` illustre une défaillance critique de configuration DACL due à une mauvaise isolation du chemin d'installation.

### Faits techniques

- **Chemin d'installation** : `C:\Program Files\PrintHelper`
- **Preuve technique (`icacls`)** : `BUILTIN\Users:(I)(M)`
- **Diagnostic** : le flag **(I)** indique que les droits de **Modification (M)** sont hérités du dossier parent.

Dans ce scénario, le service a été déployé sans rompre l'héritage des permissions ou sans restreindre les droits du groupe `Users`. Cette configuration permet à un utilisateur local d'écraser `PrintHelperSvc.exe`. Si ce service s'exécute avec les privilèges `SYSTEM`, le remplacement du binaire ouvre la voie à une élévation de privilèges complète au prochain redémarrage ou appel du service.

## 4. Analyse du vecteur d'exploitation

L'auditeur doit valider l'exploitabilité réelle, pas seulement constater une faiblesse théorique. Dans un environnement durci, l'attaquant doit souvent contourner des restrictions réseau et des solutions de sécurité.

### Séquence opérationnelle

1. **Préparation du payload** : compilation d'un binaire `helper.exe` de type reverse shell avec `x86_64-w64-mingw32-gcc` et `winsock2`.
2. **Injection via Base64** : en présence de restrictions réseau, le binaire peut être encodé puis reconstruit localement sur la cible pour éviter les téléchargements directs.
3. **Neutralisation des défenses** : dans un contexte de test autorisé, Microsoft Defender peut être temporairement désactivé pour valider le scénario d'exploitation.
4. **Hijacking et exécution** : l'arrêt du service permet de remplacer le binaire avant de relancer le service compromis.

**Logique d'exploitation (PowerShell) :**

```powershell
Stop-Service PrintHelperSvc

# Remplacement par le payload injecté puis décodé
Copy-Item "C:\Temp\helper.exe" "C:\Program Files\PrintHelper\PrintHelperSvc.exe" -Force

Start-Service PrintHelperSvc
```

Le SCM charge alors le binaire malveillant avec les privilèges `NT AUTHORITY\SYSTEM`.

## 5. Stratégies de remédiation et durcissement

La sécurisation des services doit être structurelle et reposer sur le principe du moindre privilège.

### Directives impératives de remédiation

- **Correction des DACL** : le groupe `Users` doit être limité aux droits **Lecture et Exécution (RX)**. Toute permission de **Modification (M)** ou **Écriture (W)** sur un binaire de service doit être traitée comme une vulnérabilité critique.
- **Rupture de l'héritage** : lors de l'installation de services tiers, les permissions héritées depuis la racine du disque doivent être revues ou désactivées.
- **Ownership maîtrisé** : les dossiers sensibles doivent être possédés par `TrustedInstaller` ou par le groupe `Administrators` selon le besoin opérationnel.
- **Surveillance active via SACL** : l'audit de l'accès aux objets est essentiel pour détecter les remplacements de binaires.

**Configuration de l'audit système :**

```powershell
# Activation de l'audit du File System
auditpol /set /subcategory:"File System" /success:enable /failure:enable

# Vérification de la politique d'accès aux objets
auditpol /get /category:"Object Access"
```

Un audit régulier via `icacls`, combiné à une surveillance des modifications de fichiers via les SACL, constitue une défense robuste contre l'escalade de privilèges par détournement de services Windows.