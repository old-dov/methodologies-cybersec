# Analyse Mémoire d'une Compromission PowerShell : Cas SkyLink Designs

## 1. Synthèse de l'incident

Une employée junior de l'agence web SkyLink Designs télécharge sur un forum non vérifié un script présenté comme un outil d'automatisation destiné à « nettoyer » son poste. Il s'agit en réalité d'un script PowerShell malveillant. Peu après son exécution, le poste présente un comportement anormal, ce qui conduit l'équipe sécurité à capturer un dump mémoire pour analyse.

| Élément | Valeur |
|---|---|
| **Pièce analysée** | `SKYLNK_WS_404.raw` (dump RAM, ~6,1 Go) |
| **Outil de capture** | go-winpmem (amd64 1.0-rc1) |
| **Outil d'analyse** | Volatility 3 Framework 2.26.2 |
| **Script malveillant** | `cleaning.ps1` |

L'investigation confirme l'exécution du script via PowerShell avec contournement de la politique d'exécution (`-ExecutionPolicy Bypass`).

## 2. Méthodologie et environnement

Toute commande Volatility 3 suit la forme :

```
python vol.py -f <dump> <plugin>
```

Au premier lancement, Volatility reconstruit sa table de symboles Windows (fichiers PDB) ; les exécutions suivantes sont mises en cache et nettement plus rapides.

## 3. Identification du système

```
python vol.py -f SKYLNK_WS_404.raw windows.info
```

| Variable | Valeur |
|---|---|
| NtMajorVersion / NtMinorVersion | 10 / 0 |
| Build | 15.26100 |
| Architecture | 64 bits |
| SystemTime (capture) | 2025-05-16 14:59:11 UTC |

**Point de vigilance forensique** : la clé de registre `Microsoft\Windows NT\CurrentVersion` indique `ProductName = Windows 10 Pro`, alors que le build 26100 correspond en réalité à Windows 11 (24H2). `ProductName` est un faux ami connu — Microsoft le laisse souvent figé sur « Windows 10 » sur des systèmes Windows 11. Toujours recouper avec le numéro de build plutôt que de se fier à ce seul champ.

```
python vol.py -f SKYLNK_WS_404.raw windows.registry.printkey --key "Microsoft\Windows NT\CurrentVersion"
```

## 4. Inventaire des processus PowerShell

Le plugin `windows.pslist` filtré sur « powershell » révèle 4 processus actifs au moment de la capture. Le croisement des lignes de commande (`windows.cmdline`) et de la filiation parent/enfant (`windows.pstree`) permet de les qualifier.

```
python vol.py -f SKYLNK_WS_404.raw windows.cmdline | findstr /i powershell
python vol.py -f SKYLNK_WS_404.raw windows.pslist | findstr /i powershell
python vol.py -f SKYLNK_WS_404.raw windows.pstree
```

| PID | PPID | Création (UTC) | Ligne de commande / observation |
|---|---|---|---|
| 2156 | 438 | 14:58:35 | `powershell.exe` (sans argument) — console normale |
| 2608 | 5400 | 14:57:36 | `powershell.exe` lancé par Windows Terminal (OpenConsole) |
| **10452** | **2608** | **14:58:23** | **`-ExecutionPolicy Bypass -f .\cleaning.ps1`** |
| **6480** | 4688 | 14:57:14 | **`-NoExit -Command "echo JEDHA{...}"`** |

## 5. Identification des processus suspects

Trois processus constituent la chaîne d'exécution malveillante :

- **PID 2608** — PowerShell parent, lancé depuis Windows Terminal. Ligne de commande nue, mais processus parent (PPID) de celui ayant exécuté le script malveillant : point de départ de l'action de l'utilisatrice.
- **PID 10452** — processus enfant de 2608. Exécute `cleaning.ps1` avec `-ExecutionPolicy Bypass`, contournant la politique de sécurité PowerShell. C'est le cœur de l'attaque.
- **PID 6480** — processus PowerShell exécuté avec `-NoExit -Command` affichant le marqueur laissé par l'attaquant. Il matérialise la signature de l'intrus.

**Chaîne d'exécution reconstituée** : Windows Terminal (OpenConsole, PID 5400) → PowerShell (PID 2608) → `cleaning.ps1` (PID 10452, bypass).

## 6. Récupération de l'artefact

L'artefact laissé par l'attaquant a été récupéré directement dans la ligne de commande du processus PID 6480, sans extraction supplémentaire ni décodage — il était passé en clair via l'argument `-Command "echo ..."`. Ce point illustre une règle générale de l'analyse mémoire : **la ligne de commande complète d'un processus est souvent visible en clair dans le dump**, même pour des données que l'attaquant pense éphémères.

## 7. Conclusion et cartographie MITRE ATT&CK

| Technique MITRE ATT&CK | Description |
|---|---|
| **T1059.001 — PowerShell** | Exécution de commandes via l'interpréteur PowerShell |
| **T1562 — Impair Defenses** | Contournement de la politique d'exécution (`-ExecutionPolicy Bypass`) |
| **T1204 — User Execution** | Exécution par l'utilisateur d'un fichier malveillant téléchargé |

### Recommandations

- Restreindre les droits de téléchargement et d'exécution de scripts pour les profils non techniques.
- Appliquer une politique d'exécution PowerShell restrictive (`AllSigned` / `Restricted`) et activer le **Script Block Logging**.
- Sensibiliser les équipes au risque des « outils » téléchargés sur des forums non vérifiés.
- Déployer un EDR capable de détecter les invocations PowerShell avec contournement de politique.

L'incident illustre qu'une chaîne d'attaque complète — du vecteur initial (téléchargement) jusqu'à la charge exécutée — peut être intégralement reconstituée depuis un seul dump mémoire, sans accès au disque ni au réseau.
