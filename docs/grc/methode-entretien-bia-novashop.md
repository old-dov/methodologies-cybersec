# Conduire une Business Impact Analysis : Méthode d'Entretien et Priorisation (Cas NovaShop)

## 1. Objectif de la fiche

La [BIA Colisée Express](bia-colisee-express.md) montre le **résultat** d'une Business Impact Analysis. Cette fiche documente le **processus** pour y arriver : comment mener les entretiens métier qui produisent les données MTD/RTO/RPO, et comment déjouer l'objection la plus fréquente sur le terrain — *« tous mes processus sont critiques »*.

## 2. Étape 1 — Documentation du système

Point de départ : un entretien avec l'IT Infrastructure Manager pour cartographier l'existant à haut niveau. Pour NovaShop (plateforme e-commerce) :

- **OMS (Order Management System)** : base de données centrale (commandes, paiements, statuts) — le socle dont tout le reste dépend.
- **Services applicatifs** sur serveurs cloud.
- **Intégrations externes** : fournisseur de paiement, système d'entrepôt.
- **Intégrations internes** : support client, tableaux de bord de reporting.

Cette cartographie initiale sert de fil conducteur pour identifier qui interviewer ensuite : chaque intégration ou dépendance majeure pointe vers un métier ou une équipe à consulter.

## 3. Étape 2 — Déjouer l'objection « tout est critique »

### Le piège

Face à la question *« quel est l'impact d'une interruption de ce processus ? »*, la réponse spontanée d'un responsable métier est presque toujours *« c'est critique, on ne peut pas se permettre une interruption »*. Accepter cette réponse telle quelle produit une BIA inutilisable : si tout est classé critique, aucune priorisation n'est possible en situation de crise réelle.

### La méthode de contournement

Trois leviers, appliqués systématiquement lors des 5 entretiens métier menés pour NovaShop :

1. **Chiffrer l'impact dans le temps plutôt que dans l'absolu.** Ne pas demander « est-ce critique ? » mais *« que se passe-t-il après 1h ? après 4h ? après 24h ? »*. Cette reformulation force l'interlocuteur à visualiser une dégradation progressive plutôt qu'un jugement binaire.
2. **Exiger des conséquences concrètes**, pas des qualificatifs : perte de chiffre d'affaires chiffrée, non-conformité légale précise, indicateur de mécontentement client mesurable — jamais « ce serait grave ».
3. **Forcer un classement relatif entre processus**, pas un jugement absolu isolé. Demander « si vous deviez choisir entre restaurer X et Y en premier, lequel ? » produit une hiérarchie exploitable, là où « X est-il critique ? » ne le permet jamais.

Un quatrième réflexe complète la méthode : **vérifier systématiquement l'existence d'un contournement manuel**. S'il en existe un, le RTO réel peut être plus long que l'intuition initiale ne le suggère — un processus « impossible à interrompre » se révèle souvent tolérer plusieurs heures dès lors qu'une procédure de secours manuelle existe.

## 4. Étape 3 — Résultat : criticité chiffrée des processus

| Processus | MTD | RTO | RPO |
|---|---|---|---|
| Prise de commande en ligne | 4h | 2h | ~15 min |
| Gestion des transactions de paiement | 24h | 18h | <1h |
| Coordination entrepôt & expédition | 24h | 8h | ~1h |
| Suivi de commande (support client) | 24h | 16h | 4-6h |
| Reporting de gestion & analytique ventes | 72h | 48h | 24h |

### Logique de lecture du tableau

- **La prise de commande** a le RTO le plus court : aucun contournement manuel n'existe, et l'impact sur le chiffre d'affaires est immédiat dès l'interruption.
- **Le paiement** a un RPO quasi nul malgré un RTO relativement long (18h) : la donnée financière doit être **exacte**, pas nécessairement **rapide** à restaurer. Une reprise lente mais fiable vaut mieux qu'une reprise rapide avec des transactions incohérentes.
- **Le reporting** tolère le plus long délai (RTO 48h) : il fonctionne déjà sur les données de la veille, donc son interruption ne dégrade rien d'immédiatement visible pour le client final.

Ce contraste entre RTO et RPO selon la nature du processus (opérationnel temps réel vs financier vs analytique différé) est la preuve que la méthode a fonctionné : sans elle, ces cinq processus auraient probablement été déclarés « critiques » de façon indifférenciée.

## 5. Étape 4 — Dépendances et actifs

Chaque processus est ensuite mappé à ses actifs spécifiques et ses dépendances amont — par exemple, la prise de commande dépend du serveur web et de la base OMS, tandis que la gestion financière dépend du module de réconciliation et de la passerelle de paiement (un fournisseur externe, donc une dépendance hors du contrôle direct de l'entreprise).

## 6. Étape 5 — Séquence de restauration en six paliers

La séquence respecte la **dépendance technique avant la priorité métier perçue** — un principe déjà observé dans le cas Colisée Express :

1. Base de données OMS centrale (fondation absolue).
2. Services applicatifs cloud.
3. Serveur web et flux du fournisseur de recherche (support du palier métier suivant).
4. Module de réconciliation financière + passerelle de paiement ; module entrepôt + scanners + API transporteur.
5. Système de ticketing support.
6. Plateforme BI et export nocturne.

## 7. Conclusion

Une BIA n'est fiable que si sa méthode de collecte résiste à la tendance naturelle des métiers à tout qualifier de critique. Reformuler la question en termes de temps, exiger des chiffres plutôt que des adjectifs, et forcer un classement relatif sont les trois réflexes qui transforment une série d'entretiens subjectifs en une hiérarchie de reprise réellement exploitable en situation de crise.
