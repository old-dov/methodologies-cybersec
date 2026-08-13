# Rapport Méthodologique — Conformité RGPD & AIPD (Projet GenomePath)

1. Contextualisation et Cadre de Gouvernance (Phase 1)
Dans le cadre d'un projet de recherche biomédicale internationale traitant de données de santé et de séquences génomiques, la méthodologie de conformité RGPD s'appuie sur quatre piliers :  


Registres des Traitements (ROPA) & Bases Légales :
Une double justification juridique est requise pour le traitement des catégories particulières de données (Art. 9) :

Base générale (Art. 6) : Consentement des participants (Art. 6(1)(a)) et/ou mission d'intérêt public / intérêt légitime lié à la recherche (Art. 6(1)(e)/(f)).  


Dérogation données sensibles (Art. 9) : Consentement explicite des participants ou de leurs représentants légaux pour les mineurs (Art. 9(2)(a)) et finalités de recherche scientifique assorties de garanties appropriées (Art. 9(2)(j)).  


Gouvernance & Co-responsabilité :
Définition claire des rôles entre la Fondation Sirta (responsable de traitement principal) et l'Hôpital Saint-Luc (responsable conjoint pour le recrutement et la collecte clinique). La rédaction d'un accord écrit de co-responsabilité (Art. 26) est obligatoire pour répartir les obligations d'information et d'exercice des droits.  


Transferts Transfrontaliers de Données :

Les flux vers la Suisse (Binary Helix) et le Royaume-Uni (Dynamis) reposent sur des décisions d'adéquation de la Commission européenne.  


L'hébergement par CloudVault (serveurs dans l'UE à Francfort mais avec accès support potentiel depuis les États-Unis) constitue juridiquement un transfert vers un pays tiers, nécessitant des garanties appropriées (ex. Clauses Contractuelles Types - CCT) et des mesures de sécurité complémentaires.  


Désignation d'un DPO :
Obligatoire sous l'Article 37(1)(c) du RGPD en raison du traitement à grande échelle de données de santé et génétiques.  


2. Grille de Qualification de l'AIPD / DPIA (Phase 2)
La décision d'effectuer une Analyse d'Impact repose sur les 9 critères du Comité Européen de la Protection des Données (CEPD / EDPB) :

                        ┌──────────────────────────────────┐
                        │ CRITÈRES CEPD POUR GENOMEPATH    │
                        └─────────────────┬────────────────┘
                                          │
    ┌─────────────────┬───────────────────┼───────────────────┬─────────────────┐
    ▼                 ▼                   ▼                   ▼                 ▼
 Données         Personnes           Traitement à        Croisement de      Utilisation
Sensibles       Vulnérables         Grande Échelle          Données         Innovante
(Art. 9)      (Mineurs, Patients)  (Séquençage, 15 ans) (Clinique + ADN)  (Technologies)
Règle d'arbitrage : GenomePath remplissant 5 des 9 critères (le seuil réglementaire étant de 2), la réalisation d'une AIPD (DPIA) est une obligation légale préalable au lancement.  


3. Matrice d'Analyse des Risques & Atténuation (Phase 3)
L'évaluation des risques pour les droits et libertés des personnes physiques s'articule autour des trois scénarios majeurs de la CNIL :

Accès illégitime aux données (Confidentialité) :

Risque spécifique : Ré-identification des données génétiques pseudonymisées ou accès distant par des autorités d'un pays tiers via le support technique d'un sous-traitant (ex. CloudVault aux US).  


Sévérité : Maximale (4/4) en raison du caractère unique, immuable et héréditaire de l'ADN.

Modification non désirée (Intégrité) :

Risque spécifique : Altération des séquences génomiques ou des diagnostics cliniques impactant la validité de la recherche.

Disparition / Perte de données (Disponibilité) :

Risque spécifique : Perte irréversible de données de recherche sur une durée de conservation longue (15 ans).  


Mesures de réduction des risques
Mesures Techniques : Pseudonymisation avec table de correspondance strictement isolée chez le responsable de traitement, chiffrement au repos/en transit avec gestion souveraine des clés par la Fondation (CMEK/HSM), contrôle d'accès Need-to-Know et journalisation des logs.  


Mesures Organisationnelles : Audits des sous-traitants, procédures d'accès restreint et temporaire pour le support (Just-In-Time access), formation des chercheurs et conventions contractuelles rigoureuses.

4. Risque Résiduel & Consultation Préalable de l'Autorité (Phase 4)
Évaluation du risque résiduel
Après application du plan de mesures, la vraisemblance des scénarios d'incident est drastiquement réduite, ramenant le risque résiduel à un niveau acceptable pour lancer le programme de recherche.  


Conditions de la consultation préalable (Article 36 du RGPD)
La consultation préalable de l'autorité de contrôle (ex. la CNIL) est obligatoire uniquement si le risque résiduel demeure élevé malgré la mise en œuvre de toutes les mesures de sécurité envisagées.

Scénarios de risques persistants :

Incapacité technique à empêcher un accès distant aux données en clair par les autorités d'un pays tiers (lois extraterritoriales type Cloud Act / FISA 702).

Risque de ré-identification inévitable des patients mineurs en raison de la rareté extrême de leur pathologie, malgré la pseudonymisation.  


Principe de Responsabilité Legale (Accountability) :
L'acte de consulter le régulateur ne décharge en aucun cas le responsable du traitement de sa responsabilité légale. Même en cas d'avis ou de recommandations émis par la CNIL, la Fondation Sirta (et son co-responsable) demeure seule comptable et responsable devant la loi du respect de la conformité et de la protection des données des participants.  
