# Rapport méthodologique

**Sujet** : Création d'une instance de calcul Oracle Cloud Infrastructure (OCI) pour tests EDR-ML — contournement d'un blocage réseau lors de la création rapide d'instance
**Auteur** : Jean
**Date** : 2026-08-19

---

## 1. Contexte

Création d'une instance ARM64 (VM.Standard.A1.Flex, Always Free) sous Ubuntu 24.04 Minimal, destinée aux tests EDR-ML (nécessite une architecture non-32 bits). Blocage rencontré lors de la création rapide via le formulaire "Create Instance" : le champ **Public IPv4 address assignment** restait grisé et décoché, même après sélection d'un subnet public créé "à la volée" dans le même formulaire.

## 2. Diagnostic

- Vérification du compartiment cible (identique entre le formulaire instance et la console Networking) → écarté comme cause.
- Constat : le VCN créé via le flux rapide inline n'apparaissait pas dans `Networking > Virtual Cloud Networks` avant validation finale du formulaire.
- Conclusion : le flux de création rapide "Create new virtual cloud network" + "Create new public subnet" intégré au formulaire d'instance ne finalise pas correctement l'attribut public du subnet avant l'évaluation du champ IPv4, empêchant le déverrouillage du toggle.

## 3. Solution retenue

Création manuelle et séparée des ressources réseau, en amont du formulaire d'instance, via les pages dédiées `Networking`.

### 3.1 Création du VCN

- Formulaire : `Networking > Virtual Cloud Networks > Create VCN`
- Paramètres : bloc CIDR IPv4 dédié, résolution DNS activée, sans préfixe IPv6.

### 3.2 Création de l'Internet Gateway

- Formulaire : onglet `Gateways` du VCN > `Create Internet Gateway`
- Aucune association de route table à la création (association faite manuellement à l'étape suivante).

### 3.3 Ajout de la route vers l'Internet Gateway

- Onglet `Routing` > `Default Route Table` > `Add Route Rules`
- Règle ajoutée :
  - Target Type : Internet Gateway
  - Destination CIDR Block : `0.0.0.0/0`
  - Target : Internet Gateway créé à l'étape 3.2

### 3.4 Création du subnet public

- Onglet `Subnets` > `Create Subnet`
- Paramètres : type Regional, bloc CIDR IPv4 dédié dans la plage du VCN, Subnet Access = **Public Subnet**, DNS hostnames activé, Route Table = table par défaut (contenant la règle 0.0.0.0/0), Security List = liste par défaut du VCN.

### 3.5 Reprise du formulaire de création d'instance

- Networking > sélection de "Select existing virtual cloud network" (au lieu de création à la volée)
- Sélection du VCN et du subnet créés manuellement
- Résultat : champ "Automatically assign public IPv4 address" correctement dégrisé et activable.

### 3.6 Finalisation de l'instance

- Shape : VM.Standard.A1.Flex (1 OCPU, 6 GB RAM), Always Free-eligible
- Image : Canonical Ubuntu 24.04 Minimal aarch64
- Boot volume : taille et performance par défaut, chiffrement en transit activé, clé gérée par Oracle
- SSH : génération de paire de clés via l'assistant ("Generate a key pair for me"), téléchargement immédiat de la clé privée

## 4. Connexion SSH — incident et résolution

Tentative de connexion initiale refusée (`Permission denied (publickey)`) malgré une clé correcte, en raison de permissions Windows trop permissives sur le fichier de clé privée.

### Commandes de résolution (PowerShell)

```powershell
icacls "<chemin_clé>" /inheritance:r
icacls "<chemin_clé>" /grant:r "$($env:USERNAME):(R)"
icacls "<chemin_clé>"
icacls "<chemin_clé>" /remove "BUILTIN\Administrateurs" "AUTORITE NT\Système" "BUILTIN\Utilisateurs"
```

### Connexion finale

```powershell
ssh -i "<chemin_clé>" ubuntu@<IP_publique_instance>
```

Connexion établie avec succès après restriction des permissions au seul compte utilisateur courant.

## 5. Points clés à retenir (méthodologie HIVE)

- Le formulaire de création rapide d'instance OCI (VCN + subnet inline) présente une limitation connue empêchant l'activation de l'IP publique automatique dans certains contextes (tenancy neuf notamment).
- Solution robuste : toujours créer le VCN, l'Internet Gateway, la route et le subnet **séparément** via les pages Networking dédiées avant de lancer la création d'instance.
- Sous Windows, une clé privée SSH doit être restreinte au seul compte utilisateur (`icacls`) pour être acceptée par le client OpenSSH — les groupes hérités (Administrateurs, Système, Utilisateurs) doivent être explicitement retirés.

## 6. Statut final

Instance `edr-arm64-test` opérationnelle (état Running), accessible en SSH. Prête pour la suite des tests EDR-ML.
