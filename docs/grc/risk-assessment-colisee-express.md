# Corporate Risk Assessment : Méthodologie de Cotation (Cas Colisée Express)

## 1. Contexte et périmètre

Colisée Express, division logistique (courrier & colis), doit sécuriser son réseau national de distribution avant la **période de pointe** (novembre-décembre), durant laquelle l'entreprise traite 2 millions de colis par jour. Cette Corporate Risk Assessment (Réf. RA-CE-2024-Q4) précède et alimente directement la Business Impact Analysis associée (voir [BIA et Feuille de Route de Reprise](bia-colisee-express.md)).

**Périmètre évalué** :

- **18 centres de tri automatisés (ASC)**, traitant 90 % du trafic national.
- **Systèmes d'information** : NAVEX (cœur du tracking), TERRAIN (mobilité des livreurs).
- **Datacenters** hébergeant les algorithmes logistiques centraux.

## 2. Méthode de cotation

Chaque risque est coté sur deux axes indépendants, dont le produit donne un score de criticité :

- **Vraisemblance (Likelihood)** : de 1 (très improbable) à 5 (quasi certain).
- **Impact** : de 1 (mineur) à 5 (catastrophique).
- **Score = Vraisemblance × Impact.**

Cette approche multiplicative, standard en gestion des risques (proche d'EBIOS RM ou ISO 27005), évite le biais consistant à ne raisonner que sur l'impact seul : un risque à fort impact mais quasi impossible ne mérite pas la même priorité qu'un risque à impact modéré mais fréquent.

## 3. Risques identifiés

| ID | Risque | Vraisemblance | Impact | Score | Niveau |
|---|---|---|---|---|---|
| **R-01** | Sinistre physique (incendie/dégât des eaux) sur un ASC majeur | 2 (Peu probable) | 4 (Majeur) | 8 | MOYEN |
| **R-02** | Coupure télécom affectant les terminaux TERRAIN (45 000 livreurs) | 3 (Possible) | 4 (Majeur) | 12 | ÉLEVÉ |
| **R-03** | Fuite de données via un prestataire tiers compromis | 4 (Probable) | 5 (Catastrophique) | 20 | **CRITIQUE** |
| **R-04** | Ransomware ciblé sur NAVEX et les serveurs de tri (SORTING) | 3 (Possible) | 5 (Catastrophique) | 15 | **CRITIQUE** |

### Détail des vulnérabilités sous-jacentes

- **R-01** : forte densité de matériaux inflammables (carton), infrastructure vieillissante sur les hubs secondaires.
- **R-02** : dépendance forte aux réseaux mobiles publics, capacités de cache hors-ligne limitées sur les terminaux anciens.
- **R-03** : droits d'accès excessifs accordés aux contractants temporaires en période de pointe.
- **R-04** : architecture réseau plate (absence de segmentation IT/OT), serveurs legacy non patchés dans les hubs de tri.

## 4. Ce que la cotation révèle

Le score seul ne suffit pas à comprendre la priorité réelle : R-03 (score 20) et R-04 (score 15) sont tous deux CRITIQUE, mais pour des raisons structurellement différentes.

- **R-03** doit sa criticité à une **vraisemblance élevée** (4/5) : la vulnérabilité (droits d'accès excessifs pour des contractants temporaires) est déjà présente et activement exploitable, pas hypothétique.
- **R-04** doit la sienne à un **impact quasi maximal** (5/5) : la vraisemblance reste modérée (3/5), mais un ransomware réussi provoque un arrêt total et immédiat de l'activité (2M colis/jour bloqués), sans dégradation progressive possible.

Cette distinction oriente directement le traitement : R-03 appelle une action corrective immédiate sur la gestion des accès (cause déjà active), tandis que R-04 appelle une action préventive structurelle (segmentation réseau, patch management) pour réduire une vraisemblance qui reste, elle, maîtrisable.

## 5. Synthèse et actions requises

| Niveau | Score | Nombre | Références | Action requise |
|---|---|---|---|---|
| **CRITIQUE** | 15-25 | 2 | R-03, R-04 | Mitigation immédiate et activation du Plan de Continuité |
| **ÉLEVÉ** | 10-14 | 1 | R-02 | Plan de mitigation requis |
| **MOYEN** | 1-9 | 1 | R-01 | Risque surveillé / accepté |

**Priorité absolue** : traiter R-03 et R-04 avant le démarrage de la période de pointe, pour éviter une paralysie totale de l'activité au moment où elle est la plus critique pour le chiffre d'affaires annuel.

## 6. Conclusion

Une matrice de risques n'a de valeur opérationnelle que si elle distingue explicitement, pour chaque risque coté au même niveau, ce qui relève de la probabilité et ce qui relève de la gravité — car les deux appellent des leviers de traitement différents. C'est cette matrice, une fois validée, qui sert de base directe à la Business Impact Analysis suivante : chacun des quatre risques y devient un scénario de sinistre à chiffrer en temps de reprise.
