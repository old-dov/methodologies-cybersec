# Analyse Statique avec Ghidra

Cette fiche documente la méthodologie d'analyse statique de binaires avec Ghidra, dans le cadre d'un module Incident Management / Reverse Engineering : installation de l'environnement, navigation dans le CodeBrowser, techniques de reconstruction (chaînes fragmentées, structures C, équations de validation), patch binaire, déchiffrement XOR et cassage de hash non cryptographique sur binaires stripped.

## 1. Installation de l'environnement

### Prérequis Java

Ghidra 12.x nécessite **JDK 21 minimum**. Vérification de la version active :

```powershell
java -version
```

Installation via winget si nécessaire :

```powershell
winget install Microsoft.OpenJDK.21
```

> Après installation, redémarrer le terminal pour recharger le PATH.

### Lancement de Ghidra

Depuis l'archive **release précompilée** (pas le code source GitHub, qui nécessite une compilation Gradle) :

```powershell
.\ghidraRun.bat
```

## 2. Méthodologie d'analyse statique

### Mise en place du projet

- `File → New Project → Non-Shared Project`
- `File → Import File` (sélection du binaire cible)
- Double-clic sur le binaire importé → ouverture du CodeBrowser
- Accepter l'**Auto Analysis** à l'ouverture

### Localisation du point d'entrée

- Symbol Tree → dossier **Functions** → recherche de `entry` ou `_start`
- Alternative : `Window → Entry Points` (table dédiée des points d'entrée)
- Pour les binaires MSVC (PE) : pas de nom `main` automatique — suivre la chaîne d'appels depuis `entry` jusqu'à la fonction runtime (`FUN_xxxxxxxx`), puis repérer l'appel final correspondant à `main(argc, argv, envp)`

### Recherche de chaînes en clair

- `Window → Defined Strings` pour lister toutes les chaînes lisibles du binaire
- Filtrer/scroller pour distinguer les métadonnées ELF/PE des chaînes applicatives (messages, mots de passe en dur)

### Cross-références (XREFs)

- Clic droit sur une chaîne ou un symbole → **References → Show References To Address**
- Double-clic sur la référence pour naviguer directement vers la fonction appelante
- Renommage des variables génériques (`local_x`, `iVarX`) via la touche **L** pour clarifier le code décompilé

### Reconstruction de chaînes fragmentées

Lorsque des comparaisons caractère par caractère sont effectuées dans un ordre non séquentiel (offsets `local_XX` dispersés), mapper chaque variable à sa position réelle dans le buffer pour reconstituer la chaîne complète.

### Résolution d'équations mathématiques

Pour les boucles de calcul (multiplication/addition répétées), extraire l'équation depuis le code décompilé et la résoudre manuellement (algèbre simple) pour retrouver la valeur d'entrée attendue.

### Patch binaire (bypass de conditions figées)

Pour contourner une condition qui n'est jamais atteinte dans le flux normal d'exécution :

- Repérer l'instruction de saut conditionnel bloquante dans le Listing
- Clic droit sur l'instruction → **Patch Instruction**
- Privilégier l'**inversion de la condition de saut** (ex. `JNZ` → `JZ`) plutôt que le `NOP`, car remplacer une instruction par une instruction de taille différente en octets peut être rejeté par Ghidra
- Exporter le binaire patché : `File → Export Program` (format *Original File*)
- Rendre le binaire exécutable et le lancer :

```bash
chmod +x <binaire_patché>
./<binaire_patché>
```

### Déchiffrement XOR

Pour des chaînes obfusquées par XOR à clé fixe (un octet), extraction du tableau d'octets chiffrés et de la clé depuis le code décompilé, puis déchiffrement via script :

```python
decoded = bytes([b ^ key for b in encrypted_bytes])
```

### Reconstruction de structures C

Lorsque le code décompilé manipule des offsets de pointeurs bruts (`*(undefined8 *)(ptr + 0x18)`), reconstituer manuellement les valeurs en assemblant les écritures mémoire successives (attention aux chevauchements d'offsets) pour obtenir la chaîne finale.

### Contournement anti-debug (ptrace)

Pattern classique : `ptrace(PTRACE_TRACEME, 0, 1, 0)` suivi d'un test de la valeur de retour. Une exécution normale (hors debugger) renvoie une valeur positive/nulle et emprunte le chemin légitime — aucun patch nécessaire dans ce cas, contrairement à un contournement en présence d'un debugger actif.

### Identification d'algorithmes sur binaire stripped

Sans noms de fonctions, repérer les **constantes magiques** caractéristiques d'algorithmes connus (ex. `5381` et multiplication par `33` → DJB2 hash). Une fois l'algorithme identifié, extraire le hash cible depuis la condition de comparaison.

### Cassage de hash non cryptographique (DJB2)

DJB2 n'étant pas résistant aux collisions, une recherche par force brute (script C compilé en `-O3` pour la vitesse) sur un espace de caractères alphanumériques permet de retrouver une préimage valide :

```bash
gcc -O3 -o crack crack.c && ./crack
```

> Note : une collision valide suffit à satisfaire la condition `hash == target`, mais si le flag final est lui-même chiffré avec une clé dérivée du premier caractère du mot de passe *attendu par le développeur*, une collision avec un premier caractère différent produira un flag illisible.

**Contournement :** si le schéma de chiffrement du flag final utilise une clé XOR à un seul octet, il est possible de bypasser entièrement la recherche du mot de passe en brute-forçant directement les 256 valeurs de clé possibles sur les octets chiffrés, à la recherche d'un texte clair cohérent (ex. préfixe `AEGIS{`) :

```python
for key in range(256):
    decoded = bytes([b ^ key for b in encrypted_data])
    if decoded.startswith(b'PREFIX_ATTENDU'):
        print(key, decoded)
```

## 3. Environnement d'exécution des binaires

Les binaires ELF (Linux) ne peuvent pas s'exécuter nativement sous Windows — utilisation de **WSL** :

```bash
cd /mnt/d/chemin/vers/le/binaire
chmod +x <binaire>
./<binaire>
```

## 4. Synthèse des techniques couvertes

| Technique | Outil / méthode Ghidra |
|---|---|
| Extraction de secrets en clair | Defined Strings |
| Localisation du point d'entrée réel | Symbol Tree / Entry Points / suivi de XREFs |
| Découverte de fonctions via message d'erreur | XREFs (Show References To) |
| Reconstruction de mot de passe fragmenté | Lecture décompilée + mapping d'offsets |
| Résolution d'équation de validation | Analyse du Decompiler + calcul manuel |
| Bypass de chemin d'exécution mort | Patch Instruction + export binaire |
| Déchiffrement XOR à clé fixe | Extraction d'octets + script Python |
| Lecture de structures C complexes | Analyse manuelle des offsets pointeurs |
| Contournement anti-debug (ptrace) | Exécution en environnement non tracé |
| Identification d'algo sur binaire stripped | Repérage de constantes magiques |
| Cassage de hash non cryptographique | Brute force compilé (C) + bypass par clé XOR |
