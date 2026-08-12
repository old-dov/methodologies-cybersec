# Commutation Layer 2 : Segmentation VLAN, Trunking et Spanning Tree

## 1. Objectif

Concevoir une infrastructure de commutation qui isole logiquement plusieurs départements sur une même infrastructure physique (**VLAN**), tout en assurant la communication inter-switch (**Trunking 802.1Q**) et la résilience face aux boucles de niveau 2 dans une topologie redondante (**STP**). Les trois mécanismes sont interdépendants : un déploiement VLAN à l'échelle d'un site sans STP sur les liens redondants expose directement à une tempête de broadcast.

## 2. Segmentation par VLAN

### Plan de segmentation

| VLAN ID | Département |
|---|---|
| 10 | Administration |
| 20 | HR |
| 30 | Sales |
| 40 | IT |
| 50 | Guests |

Chaque VLAN constitue un domaine de diffusion (broadcast domain) isolé : une trame broadcast émise dans le VLAN 10 n'atteint jamais un poste du VLAN 50, même si les deux machines sont connectées au même commutateur physique.

### Configuration des ports

- **Mode accès (access)** : ports connectant directement un poste utilisateur, rattachés à un seul VLAN.
- **Mode trunk (802.1Q)** : liens inter-switch, transportant le trafic de plusieurs VLAN simultanément via un tag d'encapsulation.

```
! Exemple de configuration Trunk
interface e0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown
```

### Validation

```
show vlan brief
show interfaces trunk
```

Test de bout en bout : un ping entre deux postes du **même VLAN** situés sur des switches **différents** doit aboutir, validant à la fois l'encapsulation 802.1Q et le transport correct des trames sur le lien trunk.

## 3. Spanning Tree Protocol (STP) — gérer la redondance sans boucle

Une topologie redondante en triangle (3 switches interconnectés) crée un risque structurel : sans mécanisme de contrôle, une trame broadcast circule indéfiniment entre les switches, se multipliant à chaque saut.

### Élection du Root Bridge

Par défaut, le switch avec l'adresse MAC la plus basse est élu Root Bridge — un critère non maîtrisé par l'administrateur. En production, il est préférable de **forcer** ce rôle sur un switch cœur choisi délibérément (proximité du backbone, capacité matérielle) plutôt que de le laisser à l'aléa des adresses MAC :

```
Switch1(config)# spanning-tree vlan 10 priority 4096
```

### Rôles et états des ports après convergence

| Switch | Port | Rôle | État |
|---|---|---|---|
| Switch1 (Root) | Tous | Designated | Forwarding |
| Switch2 | vers S1 | Root Port | Forwarding |
| Switch3 | vers S1 | Root Port | Forwarding |
| Switch3 | vers S2 | Alternate | **Blocking** |

Le port en état **Blocking** sur Switch3 est la clé du mécanisme : il existe physiquement, mais STP interdit tout trafic de données à son travers tant que le chemin principal reste opérationnel. C'est ce blocage logique qui élimine la boucle physique.

### Ce qui se passe si STP est désactivé

Dans cette même topologie en triangle, désactiver STP provoque une **tempête de broadcast (broadcast storm)** quasi instantanée : les trames de diffusion circulent sans contrainte, se multiplient à chaque saut entre switches, saturent les liens physiques et consomment 100 % du CPU de chaque équipement — rendant le réseau totalement indisponible. Ce n'est pas une dégradation progressive mais un effondrement brutal.

## 4. Enjeux de sécurité au-delà du lab

- **VLAN Hopping** : deux techniques permettent à un attaquant de sortir de son VLAN d'origine — le *Switch Spoofing* (négociation frauduleuse d'un port en mode trunk via DTP) et le *Double Tagging* (encapsulation de deux tags 802.1Q pour tromper le switch). Contre-mesure : désactiver DTP (`switchport nonegotiate`) et ne jamais utiliser le VLAN natif par défaut (VLAN 1) pour du trafic de production.
- **BPDU Guard** : sur les ports d'accès (jamais censés recevoir de BPDU STP), l'activation de BPDU Guard désactive automatiquement le port si une BPDU y est détectée — protection contre l'introduction d'un switch non autorisé (rogue switch) qui tenterait de manipuler l'élection du Root Bridge.
- **Root Guard** : empêche un switch en périphérie de devenir Root Bridge, même s'il annonce une priorité plus basse — préserve l'intégrité de la topologie STP conçue par l'administrateur.

## 5. Conclusion

VLAN, trunking et STP forment un socle indissociable de la commutation d'entreprise : le VLAN isole logiquement, le trunk interconnecte les switches sans perdre cette isolation, et STP garantit que la redondance physique ajoutée pour la résilience ne se retourne pas contre le réseau sous forme de boucle. Omettre l'un des trois casse la cohérence des deux autres.
