# SNMP : de l'Exposition en v2c au Durcissement en v3

## 1. Deux angles d'un même protocole

Cette fiche croise deux exercices complémentaires sur le protocole **SNMP (Simple Network Management Protocol)** : une **énumération offensive** exploitant une configuration SNMPv2c par défaut, et une **implémentation défensive** en SNMPv3 avec authentification et chiffrement. Vus ensemble, ils illustrent précisément ce que le passage de v2c à v3 est censé corriger.

## 2. Volet offensif — énumération SNMPv2c

### Contexte

SNMPv2c authentifie les requêtes par une simple chaîne de caractères en clair, la **community string**, envoyée sans chiffrement sur le réseau. De nombreux équipements conservent la valeur par défaut `public` en lecture.

### Exploitation

```bash
snmpwalk -v 2c -c public 192.168.122.37 1.3.6.1.2.1.1
```

Cette seule commande, sans aucune authentification renforcée, a permis d'extraire :

- le système d'exploitation et la version du kernel de la cible,
- le contact administratif (`admin@jedha.co`) — une fuite d'information facilitant une attaque d'ingénierie sociale ultérieure.

### Constat

La machine cible est mal configurée : l'utilisation de la communauté par défaut `public` en lecture permet une reconnaissance complète du système sans qu'aucune tentative d'intrusion active ne soit nécessaire. Cette énumération à elle seule constitue une fuite d'information exploitable pour préparer une attaque ultérieure plus ciblée.

## 3. Volet défensif — déploiement SNMPv2c puis durcissement v3

### Pièges de configuration courants (et leur résolution)

| Symptôme | Cause | Résolution |
|---|---|---|
| Interface réseau reste à l'état DOWN, pas d'IP via DHCP | Conflit entre une section `static` mal renseignée et la section `dhcp` dans `/etc/network/interfaces` | Supprimer la section statique, ne garder que `auto ens4` / `iface ens4 inet dhcp` |
| `snmpwalk` échoue avec `Timeout: No Response` | La directive `agentaddress` dans `/etc/snmp/snmpd.conf` restreint l'écoute à `127.0.0.1` (loopback uniquement) | Remplacer par `agentaddress udp:161` pour écouter sur toutes les interfaces |
| Échec d'installation de `snmp-mibs-downloader` | Paquet obsolète/absent des dépôts par défaut | Installer uniquement `snmp` (l'outil principal), sans bloquer sur ce paquet annexe |

### Passage à SNMPv3 — authentification et chiffrement

Contrairement à v2c, SNMPv3 introduit une authentification par utilisateur et un chiffrement de la charge utile :

- **Authentification** : SHA
- **Confidentialité (chiffrement)** : AES

```bash
snmpwalk -v 3 -u admin_snmp -l authPriv -a SHA -A [auth_pwd] -x AES -X [priv_pwd] 192.168.122.228 ...
```

Le niveau de sécurité `authPriv` (authentification **et** confidentialité) élimine les deux faiblesses structurelles de v2c : la community string en clair et l'absence de chiffrement du contenu des requêtes/réponses.

## 4. Ce que le cas offensif révèle sur le cas défensif

L'énumération SNMPv2c (section 2) fonctionne précisément parce que rien n'empêche un tiers non authentifié d'interroger l'agent avec la community string par défaut, en clair. La configuration v3 (section 3) neutralise ce vecteur sur deux plans simultanément : `admin_snmp` remplace `public` comme identifiant, et `authPriv` garantit qu'même une capture réseau ne révèle ni les identifiants ni le contenu des échanges.

## 5. Recommandations de durcissement

1. **Désactiver SNMPv1/v2c en production**, ou à défaut changer systématiquement la community string par défaut (`public`/`private`).
2. **Migrer vers SNMPv3** avec niveau `authPriv` (SHA + AES au minimum) pour toute infrastructure exposée au-delà d'un LAN de confiance strict.
3. **Restreindre l'accès réseau au port 161/UDP** par ACL, en n'autorisant que les IP des stations de supervision légitimes.
4. **Limiter les OID accessibles** en lecture via des vues SNMP (`view`/`context`) plutôt que d'exposer l'arbre MIB complet.
5. **Auditer régulièrement** les équipements du parc via une énumération SNMP contrôlée (comme en section 2) pour détecter les configurations par défaut oubliées.

## 6. Conclusion

SNMP illustre un cas classique en sécurité réseau : un protocole de supervision légitime, indispensable à l'exploitation, devient un vecteur de reconnaissance gratuit dès lors que sa version historique (v2c) et sa configuration par défaut restent en place. Le passage à SNMPv3 avec `authPriv` n'est pas une option de durcissement parmi d'autres : c'est la correction directe de la faille structurelle exploitée en section 2.
