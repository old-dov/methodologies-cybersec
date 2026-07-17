# Introduction aux Vulnérabilités de Configuration Active Directory : Transformer l'Annuaire en Cible

#### 1\. Fondamentaux : Pourquoi la configuration est le maillon faible

L'Active Directory (AD) constitue le socle de l'identité numérique en entreprise. Sa conception repose sur une dualité critique : il est extrêmement  **puissant mais très flexible** . Cette souplesse, nécessaire pour s'adapter à des infrastructures variées, est paradoxalement la source de ses plus grandes faiblesses. La majorité des compromissions ne résultent pas de failles logicielles complexes ("zero-day"), mais bien d'erreurs humaines dans le paramétrage quotidien.\!IMPORTANT  **Concept Clé : La Misconfiguration**  Une "misconfiguration" (mauvaise configuration) se définit comme un réglage incorrect, faible ou incomplet. Elle transforme une fonctionnalité légitime en une vulnérabilité exploitable, offrant à un attaquant un chemin pavé par l'administrateur lui-même.L'enjeu majeur pour un administrateur réside dans l'arbitrage permanent entre la flexibilité opérationnelle (fluidité des accès) et la rigueur sécuritaire (principe du moindre privilège).Ces erreurs structurelles débutent souvent par une gestion imprécise des utilisateurs et de leurs droits d'accès au sein de l'annuaire.

#### 2\. La Gestion des Privilèges : L'Escalade Silencieuse

La compromission d'un domaine repose sur l'exploitation de droits excessifs accordés par erreur ou par omission.

##### A. Les utilisateurs sur-privilégiés

Il arrive fréquemment que des privilèges élevés soient octroyés pour une opération ponctuelle, puis oubliés. Par exemple, le compte jarjar@jedha.local peut conserver des droits d’écriture sur le groupe des Administrateurs du Domaine, créant une vulnérabilité majeure.

##### B. L'imbrication de groupes (Group Nesting)

L'imbrication permet d'organiser les permissions, mais une structure complexe masque souvent des chemins d'accès critiques.| Type d'Imbrication | Structure | Risque || \------ | \------ | \------ || **Standard** | Utilisateur  $\\rightarrow$  Groupe Local  $\\rightarrow$  Ressource | Accès contrôlé et lisible. || **Dangereuse** | C3PO  $\\rightarrow$  Help Desk  $\\rightarrow$  Operations  $\\rightarrow$  Domain Admins | L'utilisateur  **C3PO**  devient Domain Admin par héritage de groupes involontaire. |

##### C. L'abus d'ACL (Access Control Lists) et droits avancés

Les permissions AD sont régies par des ACL. Certains droits permettent une escalade immédiate :

* **GenericAll :**  Contrôle total sur un objet.  
* **WriteOwner :**  Permet de s'approprier l'objet pour en modifier les droits.  
* **WriteDACL :**  Permet de modifier les permissions pour s'octroyer des privilèges arbitraires.  
* **DCSync (Replicating Directory Changes All) :**  Permet à un compte non-DC de simuler un Contrôleur de Domaine pour extraire les hashes NTLM de tous les utilisateurs.  
* **Délégation Kerberos :**  Qu'elle soit non-contrainte (Unconstrained), contrainte ou basée sur les ressources (RBCD), une délégation mal configurée permet d'usurper l'identité d'un utilisateur tiers.\!NOTE  **Note Pédagogique :**  Comme illustré dans  **SOURCE\_IMAGE\_2** , l'onglet "Security" des propriétés de l'objet montre que l'utilisateur Jarjar Binks possède le droit "Write" sur le groupe Administrators. Un tel réglage permet à un utilisateur standard de modifier la composition d'un groupe privilégié.**Le "So What?" :**  Ces oublis permettent non seulement l'escalade, mais aussi la persistance. Un attaquant peut modifier l'objet AdminSDHolder pour créer une porte dérobée permanente, l'AD réappliquant automatiquement ces ACL malveillantes toutes les 60 minutes.Si les droits constituent le moteur de l'attaque, les protocoles d'authentification en sont le carburant.

#### 3\. Protocoles d'Authentification : Kerberos vs NTLM

L'AD s'appuie sur deux protocoles majeurs pour valider l'identité des comptes.| Caractéristique | **Kerberos** | **NTLM** || \------ | \------ | \------ || **Sécurité** | Élevée (Standard moderne) | Faible (Protocole "Legacy") || **Mécanisme** | Tickets et tiers de confiance (KDC) | Challenge / Réponse (HMAC) || **Authentification Mutuelle** | **Oui**  (Validation client/serveur) | **Non**  (Vulnérable au relais) || **Vulnérabilité Principale** | Kerberoasting / Silver & Golden Ticket | NTLM Relay / Poisoning |

##### Le flux Kerberos

Selon le schéma  **SOURCE\_IMAGE\_1** , Kerberos repose sur le  **KDC (Key Distribution Center)** , composé de l'AS (Authentication Server) et du TGS (Ticket Granting Server).

1. Le client demande un TGT (Ticket Granting Ticket) à l'AS.  
2. Le KDC utilise le secret du compte krbtgt (hash de son mot de passe) pour chiffrer le TGT. Ce compte est la clé de voûte de la confiance du domaine.  
3. Le client présente son TGT au TGS pour obtenir un ticket de service.**Pourquoi NTLM est-il toujours présent ?**  Malgré ses faiblesses, NTLM perdure pour la compatibilité avec les applications anciennes, les imprimantes ou en cas d'échec de Kerberos. Son absence d'authentification mutuelle en fait la cible prioritaire des attaques réseau.

#### 4\. Anatomie des Attaques Classiques

##### 1\. Kerberoasting

1. **Mécanisme :**  Tout utilisateur peut demander un ticket TGS pour un compte de service possédant un SPN (Service Principal Name). Ce ticket est chiffré avec le hash du mot de passe du service.  
2. Énumération des SPN via GetUserSPNs.  
3. Extraction du ticket TGS (via Rubeus).  
4. Cassage hors-ligne par dictionnaire.  
5. *Impact : Compromission du compte de service (ex: sqlsvc).*  
6. **Indicateur de compromission :**  Event ID 4769 avec un chiffrement  **RC4 (0x17)** .  
7. **Note technique :**  Cette attaque est  **totalement invisible pour les systèmes de détection réseau (SIEM)**  car elle n'utilise que des requêtes légitimes.

##### 2\. NTLM Relay & Poisoning

* **Mécanisme :**  Utilisation de Responder pour empoisonner les requêtes de résolution de noms (LLMNR/NetBIOS). Pour réussir le relais, l'attaquant doit  **désactiver les modules SMB et HTTP de Responder**  pour laisser ntlmrelayx intercepter et rejouer l'authentification vers une cible vulnérable.  
* *Impact : Usurpation d'identité et exécution de commandes à distance.*  
* **Indicateur de compromission :**  Pic de trafic broadcast (LLMNR/NBT-NS) inhabituel.

##### 3\. Pass-the-Hash (PtH)

* **Mécanisme :**  Extraction des hashes NTLM depuis la mémoire du processus lsass.exe. L'attaquant utilise ensuite ce hash directement pour s'authentifier via WinRM avec l'outil evil-winrm en utilisant l'option  **\-H** .  
* *Impact : Mouvement latéral et prise de contrôle totale sans connaître le mot de passe.*  
* **Indicateur de compromission :**  Connexion avec privilèges administratifs ( **Event ID 4672** ).Chaque vecteur exploite une faille de configuration : mots de passe faibles, signature SMB désactivée ou stockage de secrets en mémoire.

#### 5\. Surveillance, Détection et Outils d'Audit

La détection ne peut se limiter à l'analyse d'un événement isolé ; elle doit s'appuyer sur l'analyse d'une  **séquence d'événements**  suspecte.

##### Événements Windows critiques :

* **4624 / 4625 :**  Logins réussis/échoués.  
* **4672 :**  Privilèges administrateurs (lié au PtH).  
* **4769 :**  Requête de ticket de service (Rechercher la valeur  **0x17**  pour le Kerberoasting).  
* **4732 / 4756 :**  Modification de groupes sensibles.

##### Outils de référence :

* **Sysmon :**  Offre une visibilité accrue, notamment sur la création de processus (ID 1\) et l'accès à la mémoire LSASS (ID 10).  
* **PingCastle :**  Réalise un audit défensif pour identifier les scores de risque et les mauvaises configurations structurelles.  
* **BloodHound :**  Utilise la théorie des graphes pour révéler les chemins d'attaque complexes (ex: comment un utilisateur "Guest" accède au rang "Domain Admin" via des ACL successives).Cependant, la détection n'est qu'une composante de la défense. La sécurité réelle passe par une remédiation structurelle visant à fermer définitivement les vecteurs d'attaque.

#### 6\. Stratégies de Durcissement (Hardening)

Voici un plan d'action basé sur les meilleures pratiques de remédiation :

*   **Déploiement de LAPS :**  Génère des mots de passe d'administration locale uniques par machine, neutralisant le Pass-the-Hash latéral.  
*   **Utilisation de comptes gérés (gMSA) :**  Pour les services, afin de rendre le cassage de tickets Kerberoasting mathématiquement impossible (120 caractères aléatoires).  
*   **Activation de la signature SMB :**  Mesure de sécurité obligatoire pour bloquer définitivement les attaques NTLM Relay.  
*   **Désactivation de LLMNR/NetBIOS :**  Via GPO, pour empêcher l'empoisonnement par Responder.  
*   **Protection des comptes sensibles :**  Ajout des administrateurs au groupe "Protected Users" et activation de  **Windows Defender Credential Guard** .  
*   **Durcissement Kerberos :**  Forcer l'utilisation de l' **AES**  et désactiver le RC4 pour protéger les tickets contre le cassage rapide.**Mesures à fort impact immédiat :**  Le déploiement de  **LAPS**  et l'activation de la  **signature SMB**  constituent les deux remparts les plus efficaces pour stopper instantanément les mouvements latéraux et l'usurpation d'identité.

##### Conclusion

La sécurité d'un Active Directory ne repose pas sur une accumulation d'outils, mais sur une  **vigilance constante**  et une  **configuration rigoureuse** . En comprenant que des réglages flexibles peuvent être détournés en armes, l'administrateur peut reprendre l'avantage et transformer son annuaire en une forteresse résiliente.  
