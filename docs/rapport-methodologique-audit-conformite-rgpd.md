# Rapport Methodologique : Audit de Conformite RGPD

**Projet :** Evaluation de la conformite de la plateforme DocWagon Connect

**Perimetre de l'audit :** Politique de confidentialite, Registre des Activites de Traitement (ROPA), Contrat de sous-traitance (DPA)

**Standard de reference :** Reglement General sur la Protection des Donnees (UE 2016/679)

## 1. Contexte et Objectifs de la Demarche

L'objectif de cet audit est de verifier l'alignement entre les pratiques reelles d'une plateforme de telemedecine et sa documentation legale et operationnelle. La methodologie repose sur un croisement systematique entre :

- **La realite operationnelle :** Les traitements de donnees effectivement realises par la plateforme.
- **Le cadre juridique :** Les exigences strictes des articles du RGPD.
- **La documentation interne et externe :** Les notices d'information, registres et contrats rediges par l'organisme.

## 2. Periodes et Phases d'Analyse

L'audit methodologique se deroule selon une approche sequentielle en quatre phases distinctes :

```text
[ Phase 1 : Audit de la Notice ]
               |
               v
[ Phase 2 : Audit du Registre (ROPA) ]
               |
               v
[ Phase 3 : Audit du Contrat (DPA) ]
               |
               v
[ Phase 4 : Synthese Dirigeante ]
```

## 3. Grilles d'Evaluation et Criteres d'Audit

### Phase 1 : Audit de l'Information des Personnes (Notice de Confidentialite)

**Referentiel : Article 13 du RGPD**

L'analyse consiste a verifier si la notice mise a disposition des utilisateurs respecte les obligations d'information lors de la collecte directe des donnees.

- **Criteres d'evaluation :**
- **Identification :** Presence de l'identite et des coordonnees du responsable de traitement ainsi que du Delegue a la Protection des Donnees (DPO).
- **Exhaustivite des donnees :** Adequation entre les categories de donnees collectees annoncees et les donnees reellement collectees lors de l'utilisation des services.
- **Legalite et finalites :** Precision des finalites poursuivies et association d'une base legale valide pour chaque finalite (avec attention particuliere a l'Article 9 pour les donnees de sante).
- **Transparence des flux :** Identification claire des destinataires, sous-traitants et eventuels transferts hors UE.
- **Exercice des droits :** Mention explicite des droits des personnes concernees et des modalites de reclamation aupres de l'autorite de controle (CNIL).

### Phase 2 : Audit du Registre des Activites de Traitement (ROPA)

**Referentiel : Article 30 du RGPD**

Cette etape vise a s'assurer que le registre interne reflete fidelement et exhaustivement l'ensemble des operations de traitement effectuees par l'organisme.

- **Criteres d'evaluation :**
- **Exhaustivite des entrees :** Verification que chaque traitement reel identifie dans l'architecture fonctionnelle fait l'objet d'une fiche dediee dans le registre.
- **Conformite des champs obligatoires (Art. 30-1) :**
  - Nom et coordonnees du responsable de traitement et du DPO.
  - Finalites du traitement.
  - Description des categories de personnes concernees et des categories de donnees.
  - Categories de destinataires et sous-traitants impliques.
  - Delais prevus pour l'effacement / durees de conservation des differentes categories de donnees.
  - Description generale des mesures de securite techniques et organisationnelles.
- **Coherence juridique :** Adequation des bases legales retenues au regard de la typologie des donnees (ex. interdiction de l'interet legitime comme base unique pour des donnees de sante).

### Phase 3 : Audit de la Relation de Sous-Traitance (DPA)

**Referentiel : Article 28(3) du RGPD**

Cette phase evalue si le contrat liant le responsable de traitement a ses prestataires techniques contient l'ensemble des clauses juridiques contraignantes requises.

- **Criteres d'evaluation (check-list Art. 28-3) :**
- **Instructions documentees :** Obligation pour le sous-traitant de traiter les donnees uniquement sur instructions ecrites.
- **Confidentialite :** Engagement de confidentialite du personnel autorise a traiter les donnees.
- **Securite des traitements :** Specification concrete des mesures techniques et organisationnelles (Art. 32).
- **Encadrement de la sous-traitance ulterieure :** Conditions d'autorisation prealable et d'information en cas de changement de sous-traitants de second rang.
- **Assistance au responsable de traitement :**
  - Prise en charge de l'exercice des droits des personnes concernees.
  - Procedure et delais de notification des violations de donnees personnelles (Art. 33 et 34).
  - Realisation des analyses d'impact (AIPD / DPIA).
- **Sort des donnees en fin de contrat :** Obligation explicite de suppression ou de restitution de l'ensemble des donnees.
- **Audits et controles :** Droit accorde au responsable de traitement de realiser des audits et inspections.

### Phase 4 : Matrice de Priorisation et Synthese Executive

Les ecarts de conformite identifies au cours des phases 1 a 3 sont evalues selon deux axes d'analyse d'impact :

1. **Risque reglementaire :** Niveau d'exposition a une sanction administrative ou financiere de la part de l'autorite de controle (CNIL).
2. **Risque operationnel et humain :** Impact potentiel sur les droits, libertes et la vie privee des personnes concernees en cas d'incident de securite ou d'usage abusif.

```text
                RISQUE REGLEMENTAIRE
                      Faible       Eleve
                   +-----------+-----------+
             Eleve | Priorite  | Priorite  |
RISQUE             |  Moyenne  | Maximale  |
OPERATIONNEL       +-----------+-----------+
            Faible | Priorite  | Priorite  |
                   |   Basse   |  Moyenne  |
                   +-----------+-----------+
```

**Structure du plan d'action recommande :**

- **Actions immediates (court terme / urgence) :** Correction des traitements illegaux ou a haut risque d'exposition sanctionnelle.
- **Actions d'alignement contractuel (moyen terme) :** Revision des conventions juridiques et mise en conformite des relations tiers.
- **Actions de gouvernance (long terme) :** Mise a jour des registres internes et consolidation des procedures operationnelles.
