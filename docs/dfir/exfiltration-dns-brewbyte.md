# Détection d'une Exfiltration par Tunneling DNS (Cas BrewByte Tech)

## 1. Le principe

Le DNS traverse presque toujours les pare-feux sans inspection profonde. Un malware peut en abuser comme canal de fuite caché : chaque fragment d'un fichier volé est encodé dans le sous-domaine d'une requête envoyée vers un serveur contrôlé par l'attaquant.

**Chaîne d'attaque** : malware exécuté → lecture d'un fichier sensible → découpage et encodage (hex) → requêtes DNS successives vers le C2 → réassemblage côté attaquant = donnée exfiltrée.

Cette fiche documente la méthode d'analyse d'une capture réseau (Wireshark/tshark) pour repérer, décoder et qualifier ce type de tunneling.

## 2. Repérer le protocole dominant

**Ce qu'on cherche** : une machine bureautique normale génère un trafic varié. Une capture composée quasi exclusivement de DNS est déjà une anomalie en soi.

```
Statistics → Protocol Hierarchy   (filtre d'affichage : dns)
```

**Résultat (BrewByte)** : DNS — 2076 / 2148 paquets (UDP 53), soit plus de 96 % du trafic capturé.

## 3. Identifier le serveur / domaine visé

**Ce qu'on cherche** : vers quelle IP et surtout vers quel domaine partent les requêtes. Un seul domaine racine revenant des centaines de fois signe un canal suspect.

```
dns.flags.response == 0
```

Ajouter la colonne `dns.qry.name` pour lire les noms interrogés en liste.

**Résultat (BrewByte)** : résolveur `192.168.117.2`, domaine `…oast.me` — un service **OAST/Interactsh** détourné de son usage légitime (habituellement utilisé pour détecter des interactions out-of-band en test d'intrusion).

## 4. Inspecter les sous-domaines

**Ce qu'on cherche** : le sous-domaine le plus à gauche n'est généralement pas un nom d'hôte lisible — c'est de la donnée encodée (souvent en hexadécimal ou base32/64).

```
dns.qry.name contains "oast"
```

Ce filtre isole l'intégralité du flux d'exfiltration.

**Résultat (BrewByte)** : `2d2d2d2d2d0a` → décodage hexadécimal : `0x2d` = `-` (×5) + `0x0a` = saut de ligne, soit `-----` suivi d'un retour à la ligne — le début caractéristique d'un en-tête PEM.

## 5. Décoder et réassembler

**Méthode** : extraire chaque sous-domaine dans l'ordre chronologique, dédoublonner (chaque tranche est généralement envoyée deux fois : une requête A puis une requête AAAA pour le même fragment), décoder le hexadécimal et concaténer.

```
tshark -r capture.pcap -T fields -e dns.qry.name > fragments.txt
# puis décodage hex hors-ligne et concaténation dans l'ordre
```

**Résultat (BrewByte)** : ~519 tranches réassemblées → une **clé privée OpenSSH complète**.

## 6. Qualifier l'attaque

Un fichier sensible exfiltré via un canal détourné (DNS) constitue une exfiltration de données par tunneling.

| Technique MITRE ATT&CK | ID |
|---|---|
| Exfiltration Over Alternative Protocol | T1048 |
| Application Layer Protocol: DNS | T1071.004 |

**Synthèse** : vecteur = DNS ; donnée exfiltrée = clé privée SSH ; destinataire = infrastructure C2 (`oast.me`).

## 7. Impact et remédiation

La clé privée exfiltrée doit être considérée comme intégralement compromise dès l'instant de la fuite, indépendamment de toute preuve d'usage ultérieur.

1. **Identifier le propriétaire** de la clé via le commentaire embarqué dans le blob OpenSSH décodé (résultat BrewByte : `student@97a0efa3a2db`).
2. **Révoquer immédiatement** la clé sur tous les systèmes où elle est autorisée.
3. **Régénérer** une nouvelle paire de clés.
4. **Retirer** les autorisations associées à l'ancienne clé publique (`authorized_keys`, dépôts Git, bastions, etc.).

## 8. Réflexes méthodologiques à conserver

- **Protocol Hierarchy en premier réflexe** : un protocole qui écrase tous les autres dans une capture est un signal, jamais une situation normale.
- **Signature du tunneling DNS** : volume anormal de sous-domaines uniques vers un même domaine racine, souvent longs et d'apparence aléatoire.
- **Un sous-domaine illisible est presque toujours de l'encodage** (hex, base32/64) — le décoder avant de conclure, c'est précisément là que se cache la donnée volée.
- **Dédoublonner A / AAAA** avant réassemblage : le même fragment part généralement deux fois ; l'oublier fausse la reconstruction.
- **Domaines à connaître côté terrain** : `oast.me`, `interactsh`, `burpcollaborator`, `*.dnslog.*` — services out-of-band légitimes en test d'intrusion, mais fréquemment détournés pour l'exfiltration en conditions réelles.
