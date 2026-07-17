# Glossaire Conceptuel : Les 6 Fonctions Vitales du NIST CSF 2.0

#### 1\. Introduction : La Cybersécurité comme un Cycle de Vie

En pédagogie cyber, nous enseignons que la sécurité n’est pas un produit que l’on achète et que l’on oublie, mais un  **processus vital et continu** . Imaginez la protection d'un organisme vivant : il ne suffit pas d'avoir une peau épaisse (protection) pour survivre. Il faut un cerveau pour anticiper les dangers (gouvernance), des sens pour percevoir une intrusion (détection) et une capacité de régénération pour guérir après une blessure (restauration).Le cadre NIST CSF 2.0 structure cette survie numérique autour de six fonctions interdépendantes. Ce cycle de vie garantit qu'une organisation ne se contente pas de "poser des verrous", mais développe une véritable résilience biologique. Tout commence par la direction et la stratégie, car sans impulsion centrale, les muscles techniques s'agitent dans le vide.

#### 2\. GOVERN (Gouverner) : Le Cerveau Stratégique

La fonction  **Govern**  est la grande nouveauté du NIST CSF 2.0. Elle agit comme le centre nerveux qui définit la stratégie, les responsabilités et les limites de l'organisation. Elle répond aux questions : "Qui décide ?" et "Quel risque acceptons-nous ?".

##### Éléments Clés de la Gouvernance

Catégorie Clé,Action Concrète (Inspirée de l'Audit Syntaris)  
Contexte (GV.OC),"Cartographier les missions critiques (ex: paiement, identité) via des interviews et tableurs."  
Stratégie de risque (GV.RM),"Formaliser l'appétence au risque pour passer d'un scoring ""narratif"" à des seuils chiffrés."  
Gestion des tiers (GV.SC),"Intégrer les 37 fournisseurs critiques (ex: AWS, GCP) directement dans les plans de réponse aux incidents."  
Rôles et Responsabilités (GV.RR),"Nommer un RSSI dédié, indépendant du CTO, pour arbitrer entre vitesse commerciale et sécurité."  
**Pourquoi c'est vital :**  En l'absence de gouvernance, l'organisation "vole à l'aveugle". Le risque lié à la  **Supply Chain**  (chaîne d'approvisionnement) est devenu une préoccupation de niveau "Board". L'histoire a montré, avec l'affaire  **SolarWinds** , qu'un tiers non supervisé peut devenir un cheval de Troie. Chez Syntaris, le fait que la gestion des fournisseurs soit restée manuelle constituait leur vulnérabilité la plus critique (Tier 1).**Transition :**  Une fois que le cerveau a fixé le cap, il doit identifier avec précision chaque membre et chaque cellule de l'organisme à protéger.

#### 3\. IDENTIFY (Identifier) : La Cartographie des Actifs

La fonction  **Identify**  consiste à dresser une carte exhaustive de l'écosystème numérique. On ne peut protéger ce que l'on ne voit pas.

* **Asset Management :**  Inventorier le matériel (serveurs AWS, ordinateurs), mais aussi les actifs intangibles (données biométriques, algorithmes de paiement).  
* **Risk Assessment :**  Évaluer les menaces pesant sur ces actifs, notamment pour les données sensibles (Art. 9 du RGPD).**Insight Pédagogique :**  L'utilisation d'une  **CMDB**  permet de visualiser les dépendances. L'audit Syntaris a révélé des "angles morts" typiques : le  **Shadow SaaS**  et l'absence de télémétrie sur les parcs  **Mac** . Un point de vigilance technique : le terme "GCP legacy" ne désigne pas une obsolescence de la plateforme Google Cloud elle-même, mais la présence d'environnements anciens et  **non durcis**  hébergés sur le cloud, créant une surface d'attaque béante.**Transition :**  Une fois l'inventaire terminé, l'organisation peut enfin déployer ses défenses actives.

#### 4\. PROTECT (Protéger) : Le Bouclier de Défense

La fonction  **Protect**  vise à réduire la surface d'attaque par des barrières proactives, qu'elles soient techniques ou humaines.

* **Mesures Techniques :**  Chiffrement AES-256, Authentification Multi-Facteurs (MFA).  
* **Mesures Humaines :**  Formation des employés (phishing, hygiène numérique).  
* **Résilience :**  Sauvegardes "air-gap" et gestion des correctifs ( *Patch Management* ).**Le concept du "Moindre Privilège" (Least Privilege) :**  Ce principe impose qu'un utilisateur ou un système ne dispose que des droits strictement nécessaires à sa mission. C'est le meilleur moyen de compartimenter une infection pour l'empêcher de se propager à tout le réseau.**Exemple concret :**  Chez Syntaris, le maintien en production de  **VPN Cisco ASA**  non patchés malgré des failles critiques (CVE) représente un trou dans le bouclier, accepté pour des raisons commerciales mais techniquement dangereux.**Transition :**  Même le bouclier le plus robuste peut être contourné ; il faut donc un radar capable de voir l'invisible.

#### 5\. DETECT (Détecter) : Le Radar de Surveillance

**Detect**  est la fonction de vigilance. Elle traque les "signaux faibles" et les indicateurs de compromission (IoC).

1. **Surveillance continue (SIEM, EDR) :**  Analyser les logs pour repérer des comportements anormaux.  
2. **Analyse des alertes :**  Corréler les sources pour confirmer s'il s'agit d'une simple erreur ou d'une intrusion.**Insight de synthèse :**  La rapidité est le facteur de survie n°1. L'audit de 2023 a montré qu'un SOC (centre de surveillance) limité aux heures de bureau (8h-18h) peut laisser une intrusion active sans réponse pendant  **3 jours** . Pour une fintech, l'absence de surveillance 24/7 est une porte ouverte aux attaquants qui profitent du week-end pour agir.**Transition :**  Dès que le radar confirme une menace, l'organisme doit mobiliser sa cellule d'urgence.

#### 6\. RESPOND (Réagir) : La Cellule d'Urgence

La fonction  **Respond**  est le bras armé de la gestion de crise. Elle intervient pour stopper l'hémorragie et limiter les dégâts.

* **Incident Management :**  Application des playbooks pour contenir la menace (isolation des serveurs).  
* **Analyse Forensics :**  Étudier les traces numériques pour comprendre comment l'attaquant est entré.  
* **Attentes légales :**  Répondre aux demandes de droits des personnes (DSR) ou aux enquêtes de la CNIL, une forme de réponse à un incident réglementaire.

##### Guide de Communication de Crise

Public cible,Objectif de la communication,Outil pédagogique  
Interne,Coordination technique et juridique.,Canaux hors-bande sécurisés.  
Clients,"Maintenir la confiance et limiter le ""churn"".",Messages pré-approuvés.  
Autorités (CNIL),Conformité légale (Notification sous 72h).,Registre des violations.  
**Pourquoi c'est vital :**  Un "containment" (confinement) trop lent multiplie les coûts par dix. Syntaris a appris qu'un délai de 3 jours pour valider une alerte est inacceptable dans une architecture moderne.**Transition :**  Après avoir éteint l'incendie, l'organisation doit renaître de ses cendres.

#### 7\. RECOVER (Restaurer) : Le Phénix de la Résilience

La fonction  **Recover**  ne signifie pas simplement "redémarrer". C'est l'art de revenir à la normale en étant plus fort qu'avant l'attaque.

* **Restauration sécurisée :**  Vérifier l'intégrité des sauvegardes avant de les réinjecter, pour éviter de restaurer le malware lui-même.  
* **Apprentissage (Post-Mortem) :**  Analyser les causes racines pour nourrir la fonction  **Govern** .**La boucle de résilience :**  Le "Phénix" ne se contente pas de survivre ; il utilise l'expérience de l'incendie pour améliorer ses futures barrières. L'analyse post-incident doit impérativement remonter au niveau de la gouvernance pour ajuster l'appétence au risque et les budgets de sécurité.

#### 8\. Synthèse : NIST CSF 2.0 et Conformité RGPD

Le cadre NIST 2.0 est la traduction opérationnelle des exigences du  **RGPD** . Les deux cadres se renforcent mutuellement pour garantir la souveraineté des données :

* **Identify & Protect :**  Répondent à l' **Art. 25 (Privacy by Design)**  en intégrant la sécurité dès la conception, et à l' **Art. 32 (Sécurité du traitement)**  par le chiffrement et le MFA.  
* **Detect & Respond :**  Sont les outils indispensables pour respecter l' **Art. 33** , qui impose la notification d'une violation de données sous 72 heures.  
* **Govern :**  Incarne le principe d' **Accountability**  (Responsabilité), soit la capacité de prouver que l'on pilote sa sécurité avec rigueur.**Le mot de la fin :**   **Identifier & Protéger \= Préparer.**   **Détecter, Réagir & Restaurer \= Survivre.**

