# Business Impact Analysis et Feuille de Route de Reprise (Cas Colisée Express)

## 1. De la matrice de risques à la BIA

Cette Business Impact Analysis (BIA, Réf. BIA-CE-2024-Q4) fait suite à la [Corporate Risk Assessment](risk-assessment-colisee-express.md) : chacun des quatre risques identifiés y devient un **scénario de sinistre** dont on chiffre précisément l'impact opérationnel dans le temps. Là où le Risk Assessment répond à *« quel risque, avec quelle probabilité ? »*, la BIA répond à *« si ce risque se matérialise, combien de temps peut-on tenir, et dans quel ordre reconstruire ? »*.

## 2. Scénarios évalués

| Scénario | Risque associé | État opérationnel |
|---|---|---|
| **A — Ransomware ciblé** | R-04 | Arrêt total : tri automatisé stoppé, tracking aveugle |
| **B — Coupure télécom** | R-02 | Perte du tracking temps réel et des signatures électroniques |
| **C — Compromission de données** | R-03 | Activité maintenue, mais dommage réputationnel et amendes RGPD |
| **D — Sinistre physique sur un hub** | R-01 | Délais régionaux de 24-48h, réacheminement possible |

## 3. Processus métier impactés par scénario

### Scénario A — Ransomware (impact global : paralysie totale du tri et du tracking)

| Processus | Dépendance | État |
|---|---|---|
| A1 — Tri automatisé des colis | Serveurs SORTING, automatisation industrielle | **ARRÊT CRITIQUE** — backlog de 2M colis/jour |
| A2 — Suivi de bout en bout | Système NAVEX | **AVEUGLE** — tracking indisponible pour toutes les parties |
| A3 — Gestion des algorithmes logistiques | Datacenters | **COMPROMIS** — serveurs chiffrés, optimisation impossible |

### Scénario C — Compromission de données (impact global : fuite de données)

| Processus | Dépendance | État |
|---|---|---|
| C1 — Gestion des accès prestataires (IAM) | Protocoles IAM, vulnérabilité de droits excessifs | **COMPROMIS** — permet l'exfiltration non autorisée |
| C2 — Gestion de la confidentialité des données clients | Bases de données accessibles aux prestataires | **VIOLÉ** — risque élevé d'amendes RGPD |

## 4. Métriques de criticité : MTD, RTO, RPO

| Sigle | Définition |
|---|---|
| **MTD** (Maximum Tolerable Downtime) | Durée maximale d'indisponibilité avant que l'impact ne devienne inacceptable pour l'activité |
| **RTO** (Recovery Time Objective) | Délai cible pour restaurer le processus à un niveau opérationnel |
| **RPO** (Recovery Point Objective) | Perte de données maximale acceptable |

### Évaluation par processus

| Processus | MTD | RTO | RPO | Justification |
|---|---|---|---|---|
| **C1/C2 — Accès & confidentialité** | N/A | **0h** | N/A | Tolérance zéro : exfiltration active de données, amendes RGPD (jusqu'à 4 % du CA) |
| **A1 — Tri automatisé** | 24h | 4h | <1h | Arrêt = 2M colis/jour bloqués, paralysie opérationnelle |
| **A3 — Algorithmes logistiques** | 24h | 4h | <4h | Prérequis technique du tri : sans lui, A1 ne peut pas fonctionner |
| **A2 — Tracking de bout en bout** | 48h | 12h | <2h | Impact majeur sur la confiance client |
| **B1 — Livraison dernier kilomètre** | 72h | 24h | 24h | Livraison manuelle possible mais fortement dégradée |
| **B2 — Preuve de livraison** | 72h | 24h | 24h | Preuve légale retardée |
| **D2 — Réacheminement inter-hub** | 5 jours | 24h | N/A | Mesure de contingence, tolère un mode dégradé prolongé |
| **D1 — Opérations d'un hub régional** | 5 jours | 48h | <24h | Impact régional seulement, le réseau absorbe la charge |

### Ce que révèle le tableau

Les processus C1/C2 imposent un RTO de **0 heure** malgré un enjeu non directement opérationnel (pas d'arrêt de production) : la logique n'est pas la vitesse de reprise technique, mais la nécessité de **stopper une fuite active** avant tout autre chose. À l'inverse, A1/A3 tolèrent 4h de RTO malgré un impact opérationnel massif : la différence entre les deux n'est pas la gravité, mais la nature de l'urgence — arrêter un dommage en cours (C1/C2) versus restaurer un service (A1/A3).

## 5. Priorisation en trois paliers

**Palier 1 — Sécurité & conformité** (C1, C2) : tolérance zéro pour l'exfiltration active ou la violation réglementaire. Doit être traité immédiatement (RTO 0h), avant même le début de la reprise opérationnelle — il s'agit d'arrêter l'hémorragie, pas de la soigner.

**Palier 2 — Opérations cœur** (A1, A2, A3, B1, B2) : le cœur battant de l'activité pendant la période de pointe. Une pause courte (24-72h) est absorbable, mais au-delà, le backlog devient irrécupérable.

**Palier 3 — Contingence régionale** (D1, D2) : la structure en réseau permet l'équilibrage de charge ; une défaillance régionale (D1) est absorbée par le réacheminement (D2), d'où un MTD plus long (5 jours).

## 6. Cartographie des dépendances (extrait)

La restauration doit respecter les dépendances techniques amont, pas seulement l'ordre de priorité métier :

- **A1 (Tri automatisé) dépend de A3** (algorithmes logistiques) : les machines de tri ont besoin des instructions de routage pour diriger les colis vers la bonne trémie.
- **A2 (Tracking) dépend de A1** : le suivi s'appuie sur les événements de scan générés par les machines de tri.
- **B1/B2 (Livraison) dépendent de A1** : les livreurs ne peuvent livrer que des colis déjà triés et expédiés vers les centres de distribution locaux.

Cette chaîne de dépendances signifie qu'il est techniquement impossible de restaurer A2 avant A1, même si l'un et l'autre étaient jugés de priorité équivalente sur le seul critère métier.

## 7. Feuille de route de reprise séquentielle

1. **T+0 — Confinement immédiat** : isoler les systèmes IAM et les bases de données clients pour clore la brèche de sécurité et satisfaire les exigences RGPD.
2. **T+4h — Redémarrage central** : restaurer les algorithmes logistiques et le réseau backbone pour rétablir la logique système.
3. **T+4h à T+12h — Redémarrage industriel** : remettre en service les serveurs SORTING des 18 ASC pour reprendre le traitement physique.
4. **T+12h — Restauration de la visibilité** : remettre NAVEX en ligne pour informer les clients du statut de leurs colis.
5. **T+24h — Activation terrain** : synchroniser les terminaux TERRAIN pour permettre la livraison finale et la capture des signatures.
6. **T+24h à T+5 jours — Contingence régionale** : restaurer les installations du hub régional ou activer le réacheminement inter-hub.

## 8. Conclusion

Cette BIA illustre un principe central de la continuité d'activité : l'ordre de restauration ne suit ni la seule gravité métier, ni la seule urgence sécuritaire, mais la combinaison des deux sous contrainte de dépendance technique. Confiner une fuite de données avant de relancer la production, puis relancer les fondations techniques avant les couches applicatives qui en dépendent, n'est pas une question de préférence — c'est la seule séquence qui fonctionne.
