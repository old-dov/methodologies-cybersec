# Référentiel Technique : Architecture et Automatisation des Group Policy Objects (GPO)

Ce document définit les standards d'architecture et les protocoles de gouvernance pour le déploiement et la gestion des politiques de groupe au sein d'un environnement Active Directory Domain Services (AD DS). Il s'adresse aux ingénieurs systèmes et responsables de la sécurité informatique pour garantir la cohérence technique et la résilience du parc.

#### 1\. Fondations de l’Infrastructure et Rôle d’Active Directory

Active Directory constitue le « cerveau » du réseau Windows. En tant que système central d'identité et de gestion, il assure l'authentification (Kerberos, NTLM) et le service d'annuaire (LDAP). Une structure logique rigoureuse est la condition  *sine qua non*  de la stabilité du système : elle permet de définir avec précision l'identité, les droits et l'appartenance de chaque entité.L'architecture AD repose sur des composants hiérarchiques et fonctionnels critiques :

* **Structure Logique :**  La  **Forêt**  (frontière de sécurité ultime), les  **Arbres**  (hiérarchies DNS) et les  **Domaines**  (unités de gestion possédant leurs propres politiques).  
* **Global Catalog (GC) :**  Élément vital contenant une copie partielle en lecture seule de tous les objets de la forêt, facilitant les recherches globales et l'accélération des processus de logon.  
* **Sites et Réplication :**  Les  **Sites**  représentent la topologie physique. Ils optimisent le trafic de réplication (intra-site rapide vs inter-site planifiée) et dirigent le trafic d'authentification vers les contrôleurs de domaine les plus proches.  
* **Contrôleurs de Domaine (DC) :**  Serveurs critiques hébergeant le fichier ntds.dit (base d'identité) et le répertoire SYSVOL.**Note d'architecture sur le stockage GPO :**  Un GPO n'est pas un objet unique. Il se compose du  **Group Policy Container (GPC)** , stocké dans la base AD, et du  **Group Policy Template (GPT)** , stocké dans SYSVOL. Une désynchronisation entre ces deux éléments compromet l'application de la politique.L'organisation interne s'appuie sur les  **Unités d'Organisation (OU)** . Elles sont le niveau privilégié pour l'application des GPO et la délégation d'administration.  
* **Avertissement de gouvernance :**  Les OU ne constituent pas des frontières de sécurité, mais des outils d'organisation et de délégation.Cette structure logique sert de socle technique à l'application des politiques de groupe.

#### 2\. Architecture de Déploiement : Hiérarchie et Ordre d’Application (LSDOU)

La prévisibilité du système dépend de la maîtrise de l'ordre de priorité des GPO. Sans une compréhension stricte de la résolution des conflits, la configuration du parc peut devenir instable ou vulnérable.L'ordre d'application suit la séquence  **LSDOU** , où le dernier paramètre appliqué prévaut sur les précédents :

1. **L (Local) :**  Politiques définies sur la machine elle-même.  
2. **S (Site) :**  Politiques liées à l'emplacement physique (Site AD).  
3. **D (Domain) :**  Politiques globales appliquées à l'échelle du domaine.  
4. **OU (Organizational Unit) :**  Politiques les plus granulaires.**Exceptions structurelles :**  L'administrateur peut utiliser l'option  **"Enforced"**  (Forcé) pour empêcher l'écrasement d'une politique par une règle descendante, ou le  **"Block Inheritance"**  (Bloquer l'héritage) pour isoler une OU des politiques parentes.| Caractéristique | Configuration Ordinateur (Computer) | Configuration Utilisateur (User) || \------ | \------ | \------ || **Cible** | Machine (indépendamment de l'utilisateur). | Compte utilisateur (indépendamment du poste). || **Application** | Démarrage / Arrêt du système. | Ouverture / Fermeture de session. || **Impact Registre** | **Registry Tattooing**  (persistance des clés). | **Registry Tattooing**  (persistance des clés). || **Exemples** | Sécurité système, Firewall, Mise à jour. | Bureau, Scripts de session, Préférences IE. |

Le "Registry Tattooing" est un risque architectural : si un GPO classique est supprimé, le paramètre reste souvent inscrit dans le registre. Cette rigidité impose une transition vers des mécanismes plus souples.

#### 3\. Optimisation de la Configuration via les Group Policy Preferences (GPP)

Les  **Group Policy Preferences (GPP)**  représentent une évolution vers une gestion moins coercitive et plus contextuelle. Elles permettent d'affiner l'expérience utilisateur tout en réduisant la complexité des scripts.Les différenciateurs majeurs par rapport aux GPO classiques sont :

* **Non-coercition et Réversibilité :**  Contrairement au "tattooing", les GPP offrent l'option technique d'annuler le paramètre dès que l'objet n'est plus dans le champ d'application ( *"Remove this item when it is no longer applied"* ).  
* **Ciblage au niveau de l'élément (Item-level targeting) :**  Une précision chirurgicale basée sur l'OS, l'adresse IP, l'appartenance à un groupe ou même la langue, sans multiplication des GPO.**Capacités opérationnelles des GPP :**  
* **Mappage de lecteurs (Drive Maps) :**  Connectivité réseau dynamique.  
* **Configuration d'imprimantes :**  Déploiement basé sur l'emplacement physique.  
* **Variables d’environnement & Registre :**  Modification granulaire sans scripting complexe.  
* **Tâches planifiées et Raccourcis :**  Automatisation des outils métiers.Cette flexibilité permet de réserver les GPO classiques au durcissement pur du système et à la gestion des accès.

#### 4\. Sécurisation du Parc et Gouvernance des Accès (Modèle AGDLP)

Les GPO sont le bras armé du  *Hardening*  système. Une  **Baseline de Sécurité**  doit être appliquée pour réduire la surface d'attaque.

##### Baseline de Sécurité (Hardening)

1. **Restrictions d'interface :**  Désactivation de CMD, PowerShell (pour les non-admins), Regedit et du Gestionnaire des tâches.  
2. **Contrôle des terminaux :**  Blocage des ports USB/médias amovibles et activation forcée de Windows Defender.  
3. **Contrôle d'accès et Identité :**  Complexité des mots de passe, verrouillage de session automatique, message légal d'avertissement et blocage de l'énumération des comptes.

##### Gouvernance des Accès : Le Modèle AGDLP

Pour une gestion évolutive et auditable, l'application du modèle AGDLP est impérative :

* **A (Accounts) :**  Les comptes utilisateurs sont regroupés dans...  
* **G (Global groups) :**  Représentant les  **Rôles Métiers**  (ex: "Service Comptabilité"), imbriqués dans...  
* **DL (Domain Local groups) :**  Représentant l' **Accès aux Ressources**  (ex: "Accès Serveur Fichiers Compta"), auxquels sont appliquées les...  
* **P (Permissions) :**  Droits NTFS ou droits de partage.Ce modèle sépare la logique métier de la gestion technique des ressources, facilitant grandement la maintenance.

#### 5\. Standardisation et Automatisation via PowerShell

L'automatisation est le garant de l'intégrité du parc. L'utilisation du module GroupPolicy permet d'industrialiser les cycles de vie des configurations et d'éliminer l'aléa humain.

##### Directives d'Automatisation et de Maintenance

* **Version Control & Rollback :**  Utilisation systématique de Backup-GPO avant toute modification majeure. La résilience repose sur une stratégie de sauvegarde et de restauration (Restore-GPO) testée.  
* **Audit et Troubleshooting :**  La commande Get-GPResultantSetOfPolicy doit être l'outil de référence pour générer des rapports de conformité et vérifier quels paramètres sont réellement appliqués sur un poste cible.  
* **Gestion du cycle de vie :**  
* New-GPO / New-GPLink : Standardisation du provisionnement.  
* Invoke-GPUpdate : Déploiement immédiat des politiques critiques à distance.

##### Nomenclature et Hygiène

Chaque GPO doit suivre une nomenclature explicite (ex: SEC-W10-Hardening-USB-Block). La documentation des modifications doit être intégrée dans les rapports générés via PowerShell pour garantir une traçabilité totale.

##### Conclusion

Une infrastructure GPO performante repose sur l'équilibre entre la rigueur de la  **structure LSDOU** , la souplesse contextuelle des  **GPP**  et une gouvernance des accès basée sur le  **modèle AGDLP** . L'adoption de PowerShell pour la maintenance transforme la gestion des politiques de groupe d'une tâche manuelle à un processus industriel sécurisé et auditable.  
