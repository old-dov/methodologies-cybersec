# pfSense : Déploiement Pare-feu et VPN Site-à-Site (WireGuard)

## 1. Objectif

Déployer une infrastructure pare-feu **pfSense** dans un environnement virtualisé (GNS3), puis étendre cette base à une interconnexion sécurisée entre deux sites distants via un tunnel **VPN Site-à-Site WireGuard**. Cette fiche couvre les deux étapes dans l'ordre où elles doivent être abordées : d'abord un pare-feu fonctionnel sur un site isolé, ensuite l'interconnexion entre deux instances.

## 2. Partie 1 — Déploiement pare-feu de base

### Architecture

| Composant | Rôle |
|---|---|
| **Nœud NAT** | Fournit l'accès Internet sortant tout en isolant la topologie dans un sous-réseau privé (préféré à un nœud Cloud pour garantir l'isolement des ressources) |
| **pfSense-1** | Pare-feu central : interface **WAN** (`em0`, DHCP client vers le NAT) et interface **LAN** (`em1`, IP statique `192.168.1.1/24`) |
| **Webterm-1** | Client de test sur le réseau local (accès via VNC) |

### Étapes de configuration

1. **Conception de la topologie** : placement des nœuds dans GNS3, connexion des interfaces.
2. **Installation** : partitionnement automatique (UFS), configuration clavier par défaut.
3. **Configuration réseau (CLI)** : assignation WAN/LAN, activation du service **DHCP** sur l'interface LAN pour l'adressage automatique des clients.
4. **Validation depuis le client** : test de connectivité (`ping`), puis accès à l'interface d'administration web via navigateur.

### Validation

- Connectivité confirmée (`ping` réussi).
- Interface d'administration accessible via `http://192.168.1.1`.
- pfSense agit correctement en tant que routeur et serveur DHCP.

Cette base — pare-feu fonctionnel, routage et DHCP opérationnels — est le prérequis avant d'introduire toute règle de filtrage avancée ou tunnel VPN.

## 3. Partie 2 — VPN Site-à-Site avec WireGuard

### Objectif

Interconnecter deux instances pfSense représentant deux sites distants, pour permettre la communication inter-LAN de façon transparente et chiffrée.

### Architecture réseau

| Site | LAN |
|---|---|
| Site 1 (pfSense-1) | 192.168.1.0/24 |
| Site 2 (pfSense-2) | 192.168.2.0/24 |
| Tunnel | WireGuard, interface OPT1 |

### A. Mise en place du tunnel WireGuard

Configuration des interfaces WireGuard sur les deux routeurs, avec échange des clés publiques. Le succès du **handshake** confirme l'établissement de la connectivité de base du tunnel — c'est le premier point de validation avant tout routage applicatif.

### B. Routage et passerelle

Une gateway dédiée (`WG_Gateway`) est créée sur chaque site, pointant vers l'IP opposée du tunnel :

| Site | Route statique |
|---|---|
| pfSense-1 | `192.168.2.0/24` via `10.0.0.2` |
| pfSense-2 | `192.168.1.0/24` via `10.0.0.1` |

### C. Filtrage pare-feu sur l'interface tunnel

Sans règle explicite d'ouverture sur l'interface OPT1 (WireGuard), le trafic inter-site reste bloqué même si le tunnel est établi — le handshake WireGuard ne suffit pas, pfSense filtre le trafic applicatif indépendamment de l'état du tunnel.

| Action | Protocole | Source | Destination |
|---|---|---|---|
| Pass | Any | Any | Any |

*(règle de démonstration en lab — à restreindre aux flux réellement nécessaires en production, voir section 5)*

### D. Validation

- **Handshake** : état actif.
- **Connectivité** : ping réussi entre les deux réseaux LAN (`192.168.1.1` ↔ `192.168.2.1`).

## 4. Point méthodologique : deux couches de contrôle distinctes

Un tunnel VPN établi (handshake actif) ne garantit pas la circulation du trafic : WireGuard sécurise et achemine les paquets au niveau du tunnel, mais **pfSense applique ses propres règles de pare-feu sur l'interface virtuelle du tunnel** comme sur n'importe quelle autre interface. Diagnostiquer un VPN Site-à-Site « qui ne passe pas » nécessite donc de vérifier séparément l'état du tunnel (handshake) et les règles de filtrage sur l'interface concernée.

## 5. Durcissement au-delà du lab

- **Restreindre la règle de filtrage** sur OPT1 aux ports et protocoles réellement nécessaires entre les deux sites, plutôt que `Any/Any`.
- **Limiter les pairs WireGuard autorisés** par clé publique explicite (WireGuard ne fait confiance qu'aux clés configurées, mais toute règle pare-feu trop permissive derrière le tunnel élargit inutilement la surface).
- **Journaliser le trafic inter-site** pour disposer d'une visibilité en cas d'incident sur l'un des deux sites.
- **Séparer les VLAN/interfaces** exposées via le tunnel de celles purement locales, pour éviter qu'une compromission d'un site n'expose immédiatement l'intégralité du LAN distant.

## 6. Conclusion

Le déploiement pfSense de base établit le socle (routage, DHCP, administration) ; le VPN Site-à-Site WireGuard l'étend à une interconnexion multi-site chiffrée. La leçon opérationnelle centrale est que la sécurité d'un tel montage ne repose pas sur le seul tunnel : elle dépend autant de la règle de filtrage posée sur l'interface virtuelle qui le représente.
