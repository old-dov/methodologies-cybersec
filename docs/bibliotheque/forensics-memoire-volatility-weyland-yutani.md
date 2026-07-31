# Forensics Mémoire avec Volatility 2.6 : Analyse d'un Dump RAM (Cas Weyland-Yutani)

Cette fiche documente la méthodologie d'analyse d'un dump mémoire (`memdump.mem`) via Volatility Framework 2.6 (standalone Windows), dans le cadre d'un lab DFIR : reconstitution de l'activité d'un attaquant ayant compromis un serveur web, puis corrélation optionnelle avec une image disque via FTK Imager.

## Contexte outillage

Volatility installé en version standalone Windows (`.exe`), non ajouté au PATH.
Appel systématique via chemin complet, avec `&` en PowerShell pour gérer les
espaces dans le chemin.

```powershell
$VOL = "C:\Users\muzee\Downloads\volatility_2.6_win64_standalone\volatility_2.6_win64_standalone\volatility_2.6_win64_standalone.exe"
$MEM = "D:\Jedha Bootcamp Cyber Lead\5.Incident Management\Weyland-Yutani\dfir-case1\memdump\memdump.mem"
```

## Partie 1 — Reconnaissance système

### Identification du profil

```powershell
& $VOL -f $MEM imageinfo
```

### Liste des processus (vue plate)

```powershell
& $VOL -f $MEM --profile=Win2008SP1x86 pslist
```

### Arborescence des processus (vue hiérarchique)

```powershell
& $VOL -f $MEM --profile=Win2008SP1x86 pstree
```

## Partie 2 — Activité de l'attaquant

### Récupération de l'historique des commandes

```powershell
& $VOL -f $MEM --profile=Win2008SP1x86 cmdscan
```

### Identification du compte associé à un processus (SID)

```powershell
& $VOL -f $MEM --profile=Win2008SP1x86 getsids -p 1972
```

## Partie 3 — Analyse approfondie et synthèse

### Dump mémoire d'un processus spécifique

```powershell
& $VOL -f $MEM --profile=Win2008SP1x86 memdump -p 2880 --dump-dir .
```

### Extraction des chaînes lisibles (alternative PowerShell à `strings`)

`strings` (Sysinternals) non disponible sur le poste. Extraction via lecture
binaire brute et regex sur les octets imprimables, encodage ISO-8859-1 pour
préserver l'alignement octet-à-octet :

```powershell
$bytes = [System.IO.File]::ReadAllBytes("2880.dmp")
$text = [System.Text.Encoding]::GetEncoding("ISO-8859-1").GetString($bytes)
$text | Select-String -Pattern "[\x20-\x7E]{4,}" -AllMatches | ForEach-Object { $_.Matches.Value } | Out-File -Encoding UTF8 "2880_strings.txt"
```

### Recherche de motifs dans les chaînes extraites

```powershell
Select-String -Path "2880_strings.txt" -Pattern "net.user" -SimpleMatch:$false
Select-String -Path "2880_strings.txt" -Pattern "%26%26"
```

### Scan des fichiers en mémoire

```powershell
& $VOL -f $MEM --profile=Win2008SP1x86 filescan > filescan_full.txt
```

### Filtrage du filescan (répertoire web + extensions sensibles)

```powershell
Select-String -Path "filescan_full.txt" -Pattern "htdocs" | Select-String -Pattern "\.zip|\.rar|\.7z|\.php|\.exe|\.jsp|\.asp"
```

## Méthodologie de raisonnement (réutilisable)

1. **imageinfo** → déterminer le profil avant tout autre plugin.
2. **pslist + pstree** → repérer les processus hors fenêtre de setup initiale
   et analyser leur lignée (parent = explorer.exe → session interactive ;
   parent = service web → exécution via l'application).
3. **cmdscan** → historique des commandes tapées en interactif ; filtrer les
   blocs de mémoire résiduelle (CommandCount incohérent, fragments).
4. **getsids -p <PID>** → le SID se terminant par `-500` désigne toujours le
   compte Administrateur intégré, quel que soit le nom du domaine/machine.
5. **memdump -p <PID> + extraction de strings** → pour retrouver l'activité
   d'un attaquant passée par une application (web) plutôt que par une
   console interactive ; penser à l'encodage URL des caractères spéciaux
   (`%26%26` = `&&`) dans les paramètres de formulaire.
6. **filescan** → toujours rediriger vers un fichier (sortie volumineuse) puis
   filtrer par répertoire applicatif connu et extensions à risque.

## Bonus — Corrélation avec l'image disque (FTK Imager)

Complément optionnel du lab : croiser les preuves mémoire avec l'image disque
complète (`Case1-Webserver.E01`), pour accéder aux logs Apache, aux
métadonnées de fichiers et à d'éventuels artefacts absents de la RAM.

### Installation FTK Imager (Exterro)

Le formulaire de téléchargement officiel filtre les domaines de webmail grand
public (Orange, MSN...) et n'accepte que des adresses à caractère
professionnel/institutionnel. Une adresse sur domaine personnel
(`@tech-etc.fr`) passe le filtre là où les FAI grand public sont rejetés.

Distribution reçue sous forme d'ISO — montage natif Windows avant
installation :

```powershell
$mount = Mount-DiskImage -ImagePath "C:\chemin\vers\FTKImager.iso" -PassThru
$driveLetter = ($mount | Get-Volume).DriveLetter
Get-ChildItem "${driveLetter}:\" -Recurse
```

### Ouverture de l'image disque

Dans FTK Imager : **File > Add Evidence Item > Image File**, puis sélection du
fichier `Case1-Webserver.E01` (format EWF reconnu nativement, pas de
conversion nécessaire). Les segments `.E02`, `.E03`... doivent rester dans le
même dossier que le `.E01` pour un rechargement automatique en séquence.

### Export d'un fichier pour analyse texte hors FTK Imager

Clic droit sur le fichier ciblé dans le panneau **File List** → **Export
Files...**, puis recherche dans le fichier exporté :

```powershell
Select-String -Path "access.log" -Pattern "exec"
```

### Méthodologie de corrélation mémoire ↔ disque

1. **Métadonnées de fichiers** (`Date Created` / `Date Modified` / `Date
   Accessed` dans le panneau File List de FTK Imager) pour situer un artefact
   retrouvé en mémoire (ex. webshell) dans une chronologie précise —
   information que Volatility seul ne fournit pas aussi finement.
2. **Logs applicatifs** (`xampp\apache\logs\access.log`) pour confirmer et
   enrichir une séquence d'attaque déduite de la mémoire (ex. corrélation
   entre le nombre de tentatives `POST` observées côté web et le nombre de
   commandes reconstruites via strings sur `httpd.exe`).
3. **Repérage par user-agent atypique** dans les logs (ex. `sqlmap/1.0-dev`)
   pour détecter des outils d'exploitation automatisés qui n'auraient laissé
   aucune trace en mémoire au moment du dump (fichiers webshell temporaires
   déjà supprimés par l'attaquant après usage).
4. **Dossier `[orphan]`** dans l'arborescence FTK Imager : pointe vers des
   entrées MFT dont le chemin parent est cassé, piste à explorer pour
   retrouver des fichiers supprimés récemment (non exploré dans cette
   session).

## Points de vigilance techniques

- Sous PowerShell, les chemins avec espaces nécessitent l'opérateur `&`
  (call operator) pour l'exécution d'un exécutable.
- `Get-Content -Encoding Latin1` n'existe pas nativement en PowerShell 5.1 ;
  passer par `[System.Text.Encoding]::GetEncoding("ISO-8859-1")` en .NET direct.
- Un dump mémoire de ~200 Mo se traite en quelques dizaines de secondes à
  quelques minutes avec une regex PowerShell — acceptable sans script externe.
