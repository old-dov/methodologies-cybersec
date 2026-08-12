# Protocole d'Audit Active Directory : Kerberoasting

## 1. Synthèse exécutive

Le Kerberoasting cible une faiblesse structurelle du protocole Kerberos plutôt qu'une faille logicielle : **tout utilisateur standard authentifié dans le domaine peut légitimement demander un ticket de service pour n'importe quel compte disposant d'un SPN**, sans avoir besoin d'un privilège particulier ni de savoir s'il a réellement besoin d'accéder au service visé.

Contrairement au LLMNR Poisoning, au NTLM Relay ou au Pass-the-Hash, cette attaque reste pleinement fonctionnelle même dans un environnement où **NTLM est entièrement désactivé** : elle n'exploite que le fonctionnement natif de Kerberos. Sur un cas d'audit type (domaine `TwinStar`), l'exploitation a permis de démontrer un risque critique de compromission du compte de service `sqlsvc`.

## 2. Mécanisme technique

Tout compte (utilisateur ou machine) exécutant un service dans le domaine possède un identifiant unique, le **SPN (Service Principal Name)**, qui l'associe à un service réseau (base de données, application web, etc.).

Lorsqu'un client veut accéder à ce service, le Contrôleur de Domaine (KDC) lui délivre un **TGS (Ticket Granting Service)**. La faille structurelle est là : ce ticket TGS est chiffré avec le **hash du mot de passe du compte de service**, et le KDC ne vérifie jamais que le demandeur a un besoin légitime d'accéder au service concerné.

| Élément | Rôle |
|---|---|
| **SPN** | Identifie un compte comme exécutant un service réseau |
| **TGS** | Ticket délivré par le KDC, chiffré avec le hash du compte de service |
| **KDC** | Délivre le ticket sans contrôle du besoin réel d'accès |
| **Cassage hors-ligne** | Le hash du ticket est attaqué localement, sans contact réseau avec le domaine |

## 3. Déroulement de l'exploitation

1. **Énumération des SPN** : interrogation de l'AD avec `GetUserSPNs` (Impacket) pour lister les comptes porteurs d'un SPN. Un compte de service critique type `sqlsvc` (lié à Microsoft SQL Server) constitue une cible de choix.
2. **Extraction du ticket TGS** : demande d'un ticket de service via un outil de post-exploitation Kerberos (`Rubeus.exe kerberoast`). Le KDC répond sans vérifier la légitimité de la demande.
3. **Attaque par dictionnaire hors-ligne** : transfert du hash extrait sur la machine attaquante et cassage via John the Ripper ou Hashcat combiné à une wordlist (ex. `rockyou.txt`).

```
# Énumération (Impacket)
GetUserSPNs.py twinstar.local/user:password -dc-ip <IP_DC> -request

# Extraction (Rubeus, depuis un hôte du domaine)
Rubeus.exe kerberoast /outfile:hashes.kerberoast

# Cassage hors-ligne
hashcat -m 13100 hashes.kerberoast rockyou.txt
```

Cette opération de cassage se déroule **entièrement en local**, sans aucune requête supplémentaire vers le domaine : elle est donc invisible pour les systèmes de détection réseau une fois le ticket extrait.

## 4. Détection

- **Event ID 4769** (« A Kerberos service ticket was requested ») : à surveiller en priorité.
- **Indicateur d'attaque prioritaire** : type de chiffrement du ticket demandé égal à **RC4 (valeur `0x17`)**. Les comptes de service modernes en gMSA utilisent AES ; une demande RC4 sur un compte à SPN est un signal fort.
- **Volumétrie anormale** : une avalanche de requêtes TGS émises depuis une seule machine cliente non-administratrice signe généralement l'usage d'un outil automatisé (Rubeus, Impacket).

## 5. Plan de remédiation et durcissement

Le Kerberoasting exploite le fonctionnement légitime de Kerberos : il est **impossible de bloquer les demandes de tickets**. La défense repose donc sur la robustesse des clés et la surveillance.

### A. Comptes gérés (gMSA) — la parade absolue

Remplacer les comptes utilisateurs standards utilisés pour faire tourner les services (ex. `sqlsvc`) par des **gMSA (Group Managed Service Accounts)**. Ils possèdent des mots de passe de 120 caractères générés aléatoirement, changés automatiquement tous les 30 jours : même un ticket TGS intercepté devient mathématiquement impossible à casser par brute-force.

### B. Complexité des comptes de service traditionnels

Si les gMSA ne peuvent pas être déployés, imposer une politique stricte : **25 à 30 caractères aléatoires minimum**.

### C. Durcissement du chiffrement Kerberos

Forcer exclusivement l'usage d'**AES128_HMAC / AES256_HMAC** pour les tickets, et désactiver le chiffrement hérité **RC4** dans les propriétés des comptes AD. RC4 est nettement plus rapide et facile à casser lors d'une attaque par dictionnaire.

### D. Surveillance

Configurer le SIEM pour alerter sur l'Event ID 4769, en priorisant les tickets chiffrés en RC4 (`0x17`) et les pics de requêtes TGS depuis un poste non-administrateur.

## 6. Conclusion

Le Kerberoasting rappelle qu'une architecture d'authentification robuste (Kerberos) ne protège pas contre de mauvaises pratiques de gestion des comptes de service. La bascule vers les gMSA et le durcissement du chiffrement neutralisent l'attaque à la racine, là où la détection seule ne fait qu'en limiter les conséquences.
