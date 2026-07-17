# Manuel de Procédures Opérationnelles : Réponse aux Compromissions d'Identifiants (Phishing)

#### 1\. Gouvernance et Cadre Méthodologique PICERL

Le phishing constitue le vecteur d'entrée de plus de  **90 % des cyberattaques mondiales**  (source : CISA). Face à une tentative de "Credential Harvesting", l'improvisation est proscrite. Ce manuel impose l'application rigoureuse du framework  **PICERL**  (Préparation, Identification, Confinement, Éradication, Récupération, Lessons Learned) pour réduire mécaniquement le temps d'exposition et garantir une coordination optimale entre les unités techniques et décisionnelles.

##### Rôles et Responsabilités Opérationnelles

* **Incident Commander :**  Autorité centrale coordonnant la réponse et validant les décisions stratégiques.  
* **Analyste SOC :**  Responsable du triage, de l'investigation forensique et de la corrélation des logs.  
* **Administrateur Système :**  Exécuteur des mesures techniques de restriction (AD, IAM, filtrage réseau).  
* **DPO (Data Protection Officer) :**  Garant de la conformité RGPD et responsable du dossier de notification CNIL.

##### Niveaux de Sévérité et SLAs (Service Level Agreements)

L'intervention doit impérativement respecter les objectifs de temps suivants :| Niveau de Sévérité | Critères d'Activation | SLA de Réponse || \------ | \------ | \------ || **Critique** | Compromission d'identifiants à hauts privilèges (Administrateurs, VIP). | **15 minutes** || **Élevé** | Compromission d'identifiants d'un utilisateur standard. | **1 heure** |  
La maîtrise du cycle de vie de l'incident repose sur la compréhension immédiate des méthodes de l'attaquant via l'analyse des TTPs.

#### 2\. Identification : Cartographie des TTPs et Analyse des Vecteurs

L'identification ne doit pas se limiter à un constat de "clic suspect". Elle doit s'appuyer sur la méthodologie des  **Attack Trees**  (NCSC) : identifier les risques abstraits (exfiltration de données) pour remonter aux vecteurs d'attaque concrets et aux sources de logs pertinentes.

##### Mapping MITRE ATT\&CK et Impact sur la Chaîne d'Attaque

Chaque technique identifiée nécessite une réponse ciblée :

* **T1566.002 (Spearphishing Link) :**  Vecteur d'accès initial. Utilisation de liens imitant des services tiers (ex: Docusign).  
* **T1056.003 (Web Portal Capture) :**  Capture active des identifiants via un formulaire frauduleux.  
* **T1539 (Steal Web Session Cookie) :**   **Impact critique.**  Permet le vol du cookie de session pour  **contourner l'authentification multi-facteurs (MFA)**  même si celle-ci est active.  
* **T1036 (Masquerading) :**  Évasion de défense par imitation visuelle de sites légitimes.  
* **T1583.001 (Acquire Infrastructure) :**  Enregistrement de domaines typosquattés pour  **outrepasser les filtres de réputation**  basés sur les domaines connus.

##### Indicateurs de Compromission (IOCs) et Signaux Forensiques

Catégorie,Indicateurs Concrets,Sources de Logs / Event IDs  
Réseau,"Domaines typosquattés (ex: docusign-secure.net), certificats SSL récents (\< 7j), requêtes POST vers domaines inconnus.","Logs Proxy, DNS, IDS/IPS."  
Compte,"Connexions hors horaires (3h du matin), localisations atypiques, changement de mot de passe soudain.","Event ID 4624  (Logon réussi),  4625  (Échec)."  
Email,"Échec SPF/DKIM/DMARC, URLs raccourcies, pièces jointes .html suspectes.","Email Gateway, Headers email."  
Endpoint,"Création de processus anormaux post-visite, modification du fichier hosts.","Event ID 4688  (Process creation), Sysmon."  
Cette cartographie permet de cibler précisément l'isolation lors de la phase suivante.

#### 3\. Stratégies de Confinement et Isolation Immédiate

Le confinement doit stopper net la propagation latérale. L'objectif est de verrouiller le périmètre compromis dans les  **30 premières minutes**  suivant la confirmation.

##### Procédures de Neutralisation Immédiate

1. **Isolation de l'identité :**  Désactivez immédiatement le compte dans l'Active Directory (AD) ou l'IAM (Azure AD, Okta).  
2. **Blocage périmétrique :**  Inscrivez les domaines frauduleux sur les listes de blocage DNS (Cisco Umbrella) et les Firewalls.  
3. **Communication Hors-Bande :**  Utilisez exclusivement un canal sécurisé dédié (\#incident-response) ; n'utilisez jamais la messagerie d'entreprise tant qu'elle est suspectée de compromission.

##### Checklist Opérationnelle (Timeline T+30 min)

*  Suspendre le compte utilisateur compromis.  
*  Bloquer l'accès aux URLs malveillantes au niveau du Proxy/Firewall.  
*  Mettre l'email original en quarantaine globale (Exchange Admin/Google Workspace).  
*  Notifier l'équipe de réponse via canal sécurisé (hors-bande).

#### 4\. Éradication : Neutralisation des Accès et Remédiation Technique

La réinitialisation simple du mot de passe est vaine si l'attaquant conserve des accès persistants via des tokens ou des sessions web.

##### Procédure de Purge et Révocation

* **Sessions & Tokens :**  Invalidez systématiquement toutes les sessions actives (SSO) et révoquez l'intégralité des tokens OAuth accordés.  
* **Nettoyage Messagerie :**  Purgez l'email malveillant de toutes les boîtes de réception de l'organisation via Exchange Admin ou Google Vault.  
* **Signalement :**  Déclarez le domaine frauduleux sur PhishTank, Google Safe Browsing et auprès de la CISA.

##### Guide de Décision : Traitement de la Machine Victime

Scénario,Action Requise,Justification  
Saisie d'ID sans payload,Scan EDR complet \+ Surveillance,Pas d'exécution de code détectée via logs proxy (pas de téléchargement).  
Payload détecté / Doute,Ré-indexation (Re-imaging),Risque de persistance furtive. Analyse forensique (Autopsy/Volatility) préalable recommandée.

#### 5\. Rétablissement (Recovery) et Renforcement de la Posture

La restauration doit impérativement aboutir à un état de sécurité supérieur à l'état initial (durcissement).

##### Protocoles de Restauration et de Durcissement

* **Authentification :**  Réinitialisation du mot de passe (min. 16 caractères, génération aléatoire).  
* **MFA Résistant :**  Activation obligatoire d'un MFA de type  **FIDO2 ou clé physique (YubiKey)** . Les SMS et TOTP sont jugés insuffisants face aux techniques de vol de session.  
* **Sauvegardes :**  Application stricte de la règle  **3-2-1**  (3 copies, 2 supports, 1 hors ligne). Vérifiez l'intégrité des backups avant toute réintégration.  
* **Endpoint Humain :**  Procédez à la  **re-sensibilisation immédiate**  de l'utilisateur (explication du vecteur, signes d'alerte, rappel des procédures).

##### Contrôle d'Intégrité Post-Restauration

*  Supprimer toute règle de redirection email suspecte.  
*  Vérifier l'absence de nouvelles applications tierces autorisées dans l'IAM.  
*  Maintenir un monitoring renforcé des logs d'authentification pendant 30 jours.

#### 6\. Conformité Réglementaire et Protocole de Notification CNIL

L'exposition de données personnelles transforme l'incident technique en obligation juridique majeure.

##### Protocole Article 33 du RGPD

Toute violation de données doit être notifiée à la CNIL dans un délai maximal de  **72 heures** . Le rapport doit inclure la nature de la violation, les catégories de données et les mesures de remédiation.

##### Timeline Légale et Administrative

* **T+0 à T+15 min :**  Détection et qualification du niveau de sévérité.  
* **T+24h :**  Évaluation de l'impact sur les données personnelles.  
* **T+48h :**   **Clôture technique**  de l'incident et finalisation du rapport technique.  
* **T+72h :**  Échéance limite pour la notification officielle à la CNIL.  
* **T+5 jours :**  Réunion de Post-Incident Review (PIR).

#### 7\. Retours d'Expérience (Lessons Learned) et Outillage

L'amélioration continue est l'unique rempart contre l'évolution des TTPs. La réunion de Post-Incident Review doit impérativement se tenir à  **T+5 jours** .

##### Boîte à Outils Recommandée par Phase

Phase,Outils,Usage Spécifique  
Détection,"SIEM (Splunk/Sentinel), EDR, Email Gateway.",Corrélation et détection de comportements anormaux.  
Investigation,"VirusTotal ,  URLScan.io ,  DNSTwist .","Analyse d'URL, sandboxing et détection de typosquattage."  
Analyse Email,"MXToolbox , Headers complets.",Vérification SPF/DKIM/DMARC et réputation IP.  
Forensique,"Autopsy ,  Volatility ,  Wireshark ,  Any.run .","Analyse disque, mémoire RAM, trafic et sandbox .html."  
Containment,"AD, Azure AD, Okta, Firewall.",Isolation identité et blocage de flux.  
Chaque incident clôturé doit entraîner une mise à jour systématique de ce playbook, l'enrichissement des règles de détection du SIEM et l'intégration des nouveaux IOCs dans la base de Threat Intelligence.  
