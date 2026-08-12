# Routage Dynamique Redondant avec OSPF

## 1. Objectif

Établir des adjacences **OSPF (Open Shortest Path First)** entre trois routeurs interconnectés en triangle, afin d'obtenir une topologie redondante où la convergence des routes est automatique en cas de panne d'un lien — sans intervention manuelle sur les tables de routage.

## 2. Topologie et plan d'adressage

Trois routeurs (Paris = R1, Berlin = R2, Rome = R3) interconnectés en triangle pour garantir qu'il existe toujours au moins deux chemins entre deux sites quelconques.

| Routeur | Interface | Adresse IP | Site |
|---|---|---|---|
| R1 | Fa0/0 | 192.168.12.1/24 | Paris |
| R1 | Fa0/1 | 192.168.31.1/24 | Paris |
| R2 | Fa0/0 | 192.168.12.2/24 | Berlin |
| R2 | Fa0/1 | 192.168.23.1/24 | Berlin |
| R3 | Fa0/0 | 192.168.23.2/24 | Rome |
| R3 | Fa0/1 | 192.168.31.2/24 | Rome |

## 3. Configuration OSPF

L'ensemble des liens est déclaré dans l'**Area 0 (Backbone)** — le choix le plus simple pour une topologie à trois nœuds, qui évite la complexité d'une architecture multi-aires (ABR, redistribution).

```
router ospf 1
 network 192.168.12.0 0.0.0.255 area 0
 network 192.168.23.0 0.0.0.255 area 0
 network 192.168.31.0 0.0.0.255 area 0
```

## 4. Vérification et résultats

| Vérification | Commande | Résultat attendu |
|---|---|---|
| Adjacences | `show ip ospf neighbor` | État **FULL** sur tous les liens |
| Table de routage | `show ip route` | Apparition des routes marquées **O** (OSPF) |
| Redondance | — | Chemins multiples disponibles (ECMP) entre deux sites quelconques |

L'état **FULL** confirme que les deux routeurs voisins ont synchronisé leur base de données d'état de liens (LSDB) et calculé un arbre de plus court chemin cohérent via l'algorithme SPF (Dijkstra).

## 5. Pourquoi le triangle plutôt qu'une topologie en étoile ou en ligne

Une topologie en triangle est la structure minimale démontrant la valeur d'OSPF : chaque routeur dispose de **deux chemins possibles** vers chacun des deux autres sites. En cas de coupure d'un lien (ex. Paris-Berlin), OSPF recalcule automatiquement le chemin via le troisième nœud (Paris → Rome → Berlin) sans intervention manuelle, alors qu'une route statique équivalente resterait figée sur le lien en panne.

## 6. Considérations de sécurisation (au-delà du lab)

Un déploiement OSPF en production ne doit pas s'arrêter à la convergence fonctionnelle :

- **Authentification MD5 (ou SHA sur les plateformes récentes)** sur les paquets OSPF, pour empêcher l'injection de routes falsifiées par un tiers non autorisé sur le segment.
- **Passive-interface par défaut** sur toutes les interfaces qui ne doivent pas former d'adjacence (ex. interfaces orientées utilisateurs), pour réduire la surface d'attaque aux seules interfaces inter-routeurs légitimes.
- **Filtrage des routes annoncées** (`distribute-list`) pour éviter qu'un routeur compromis ne puisse injecter des routes arbitraires dans le domaine OSPF.

## 7. Conclusion

OSPF transforme une topologie physiquement redondante en une infrastructure réellement résiliente : la redondance des liens ne vaut que si le protocole de routage sait l'exploiter automatiquement. Le triangle Paris-Berlin-Rome illustre ce principe à l'échelle minimale, directement transposable à des architectures multi-sites plus larges.
