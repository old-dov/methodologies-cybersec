# Stratégies de Traitement du Risque et Discours Comité (Cas TechnoPlast)

## 1. Objectif de la fiche

Identifier un risque ne suffit pas : il faut ensuite choisir une **stratégie de traitement**, la chiffrer, et la défendre devant un comité de direction qui arbitre entre coût de la mesure et perte évitée. Cette fiche documente cinq décisions réelles chez TechnoPlast (industrie), une pour chacun des cas les plus fréquents rencontrés en gestion des risques, structurées autour des quatre stratégies classiques : **Réduire, Éviter, Transférer, Accepter**.

## 2. Les quatre stratégies de traitement

| Stratégie | Principe | Quand l'utiliser |
|---|---|---|
| **Réduire (Mitigate)** | Diminuer la vraisemblance ou l'impact par des mesures compensatoires | Le risque est significatif mais une mesure proportionnée existe |
| **Éviter (Avoid)** | Supprimer la source du risque (décommissionnement, arrêt de l'activité concernée) | Le coût de maintien dépasse la valeur métier résiduelle |
| **Transférer (Transfer)** | Reporter la charge financière sur un tiers (assurance, contrat) | Le risque est rare mais potentiellement catastrophique, et un marché de couverture existe |
| **Accepter (Accept)** | Assumer consciemment le risque résiduel, sans investissement supplémentaire | Le coût de traitement dépasse largement la perte attendue, et le risque est déjà partiellement confiné |

Le point commun aux cinq cas ci-dessous n'est pas la stratégie choisie (elles diffèrent), mais la méthode : **chiffrer systématiquement le coût de la mesure face à la perte évitée**, jamais trancher sur la seule intuition de gravité.

## 3. Cas 1 — Faille critique sur un ERP legacy → RÉDUIRE

**Contexte** : l'ERP (gestion, stocks, production) est touché par une faille critique (CVSS 9.8) activement exploitée. Le patcher exige 2 semaines d'arrêt total de production. Une migration cloud est prévue dans 4 mois.

**Le piège à éviter** : patcher immédiatement (coûte 750 000 € d'arrêt de production) ou ne rien faire en attendant la migration (expose à un ransomware potentiellement dévastateur). Les deux options extrêmes sont mauvaises.

**Décision** : réduire le risque par des mesures compensatoires temporaires, sans toucher au code applicatif :

1. Isoler l'ERP d'Internet (fermeture de tous les accès directs).
2. Imposer un VPN renforcé par MFA pour tout accès distant.
3. Surveillance ciblée du pare-feu et des logs sur les tentatives d'exécution de code à distance (RCE) spécifiques à cette faille.
4. Sanctuariser les sauvegardes (vérification quotidienne, copies hors-ligne/immuables).

**Coût** : quasi nul (temps d'ingénierie interne). **Perte évitée** : 750 000 € d'arrêt de production + risque de ransomware.

## 4. Cas 2 — Ordinateurs commerciaux sans chiffrement de disque → RÉDUIRE

**Contexte** : 15 commerciaux terrain manipulent des données sensibles (devis, NDA, données clients) sur des disques non chiffrés.

**Le piège à éviter** : croire que la politique « Cloud-first » élimine le risque local — les fichiers téléchargés, caches navigateur et mots de passe enregistrés restent stockés physiquement sur la machine, et un effacement à distance (Intune) est inopérant si le voleur n'est jamais reconnecté à Internet.

**Décision** : activer BitLocker (chiffrement complet du disque natif Windows) via GPO/Intune sur les 15 postes sous 15 jours, avec centralisation des clés de récupération dans Azure AD.

**Coût** : 5 000 €. **Perte évitée** : jusqu'à 720 000 € d'amende CNIL (4 % du CA) + perte de couverture de l'assurance cyber (les polices excluent généralement la « négligence de sécurité »). Ratio coût/bénéfice qui rend la décision quasi automatique.

## 5. Cas 3 — Datacenter unique pour toute la production → RÉDUIRE (par transfert partiel)

**Contexte** : toute l'infrastructure de production est hébergée dans un seul datacenter. Un sinistre majeur paralyserait une usine distante, à hauteur de 50 000 €/jour de pertes.

**Le piège à éviter** : se reposer uniquement sur l'assurance cyber, qui n'intervient qu'après une franchise de 48h (soit déjà 100 000 € de pertes nettes avant tout remboursement) et ne protège pas l'image de marque.

**Décision** : arbitrage économique plutôt que technique. L'historique montre des interruptions rares et courtes (deux coupures de 4h sur l'année). Investir 200 000 € en migration cloud ou 120 000 €/an pour un second datacenter est jugé disproportionné face à la fréquence réelle constatée. Stratégie retenue : **transférer** le risque catastrophique via une police d'assurance (15 000 €/an, couverture jusqu'à 500 000 €), complétée par une mesure de réduction à bas coût (sauvegardes hors-site répliquées quotidiennement) pour absorber les pannes courtes sans attendre la franchise.

**Point méthodologique** : ce cas illustre qu'une même situation peut combiner deux stratégies — transférer le risque catastrophique rare, réduire le risque fréquent à bas coût — plutôt que de choisir une seule stratégie pour l'ensemble du problème.

## 6. Cas 4 — Portail client legacy vulnérable (PHP 5.3, 2012) → ÉVITER

**Contexte** : un portail obsolète, criblé de failles majeures (SQLi, RCE), n'est plus utilisé que par 12 clients — dont un seul (0,5 % du CA) exige son maintien pour une ancienne API.

**Le piège à éviter** : dépenser 80 000 € pour reconstruire un outil à faible valeur, ou accepter le risque au péril du réseau interne entier pour satisfaire un compte représentant 0,5 % du chiffre d'affaires.

**Décision** : décommissionnement pur. Notification officielle de fin de service aux 12 clients restants, migration accompagnée, délai de grâce de 90 jours pour le dernier client récalcitrant, puis coupure définitive du serveur et des flux réseau associés.

**Coût** : 12 000 € (migration et accompagnement). **Perte évitée** : 80 000 € de développement inutile + risque d'intrusion totale du réseau interne via une passerelle obsolète.

## 7. Cas 5 — Serveur de fichiers en fin de vie (Windows 2008 R2) → ACCEPTER

**Contexte** : un serveur d'archives, sans mise à jour depuis 2020, isolé d'Internet et consulté 2 à 3 fois par trimestre par 8 utilisateurs seulement.

**Le piège à éviter** : dépenser 45 000 € pour une migration surdimensionnée (SharePoint), ou laisser le serveur tel quel sans traiter le risque de mouvement latéral en cas d'intrusion interne.

**Décision** : acceptation formelle du risque résiduel, assortie d'une mesure de confinement à coût quasi nul plutôt que d'une migration complète.

1. Isolation réseau stricte (micro-segmentation/VLAN dédié), tous flux bloqués sauf ceux des 8 utilisateurs autorisés.
2. Formalisation écrite de l'acceptation du risque, validée et signée par le Risk Committee (CEO/CISO).
3. Revue annuelle de gouvernance pour réévaluer la pertinence du maintien.

**Point méthodologique** : l'acceptation d'un risque n'est légitime que si elle est **formalisée et signée** par une autorité de gouvernance identifiée — une acceptation tacite ou implicite n'est pas une stratégie de traitement, c'est une absence de décision.

## 8. Structure d'un discours comité efficace

Les cinq cas partagent une même mécanique rhétorique, transposable à toute présentation devant un comité de direction :

1. **Nommer le piège** : énoncer explicitement l'option intuitive mais mauvaise (patcher en urgence, tout migrer, ne rien faire) avant de la rejeter.
2. **Chiffrer les deux plateaux de la balance** : coût de la mesure vs perte financière évitée, toujours en euros comparables.
3. **Justifier la stratégie par la structure du risque**, pas par une préférence : fréquence élevée → réduire ; rareté catastrophique → transférer ; valeur résiduelle nulle → éviter ; risque déjà confiné et coût de traitement disproportionné → accepter.
4. **Terminer par un plan d'action concret et daté**, jamais par une recommandation abstraite.

## 9. Conclusion

Le choix entre réduire, éviter, transférer et accepter n'est jamais une question de prudence générale : c'est un calcul spécifique à la structure de chaque risque (fréquence, gravité, valeur métier résiduelle, existence d'un marché de transfert). Les cinq cas TechnoPlast montrent qu'une organisation mature applique les quatre stratégies simultanément sur son registre de risques, plutôt que de systématiser un unique réflexe (typiquement, mitiger tout par défaut) qui gaspille du budget sur les risques à faible enjeu et sous-investit sur les risques structurels.
