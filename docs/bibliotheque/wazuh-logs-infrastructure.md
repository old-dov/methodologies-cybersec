# Briefing : Infrastructure de Détection et Gestion des Logs avec Wazuh

#### Résumé Exécutif

Ce document synthétise les principes fondamentaux et les applications pratiques du déploiement de Wazuh pour la surveillance de sécurité et la conformité (SOC2, ISO 27001). Les points clés incluent :

* **Polyvalence de la collecte :**  Wazuh permet une centralisation des logs via des agents dédiés ou par des méthodes sans agent (Syslog), facilitant l'intégration d'équipements réseau comme pfSense ou de conteneurs Docker.  
* **Architecture de détection :**  La transformation des logs bruts en alertes exploitables repose sur un système binaire composé de  **décodeurs**  (structuration des données) et de  **règles**  (analyse de sévérité de 0 à 15).  
* **Capacités d'investigation :**  L'utilisation combinée du  **DQL**  (Dashboards Query Language) pour la recherche de logs et du  **WQL**  (Wazuh Query Language) pour le filtrage de l'inventaire assure une visibilité complète sur l'infrastructure.  
* **Validation Opérationnelle :**  Les tests effectués sur des infrastructures types (ex: Kessel Dynamics) démontrent une capacité réelle à détecter des menaces critiques, telles que des chevaux de Troie réseau (BlackSun) ou des élévations de privilèges.

#### 1\. Architecture et Environnement de Déploiement

L'implémentation d'une solution SIEM comme Wazuh s'inscrit souvent dans une stratégie de mise en conformité et de protection contre les menaces internes et externes.

##### Étude de cas : Infrastructure Kessel Dynamics

Le déploiement réalisé pour Kessel Dynamics illustre une topologie réseau standardisée (192.168.153.0/24) sur VMware :| Composant | OS | IP | Rôle || \------ | \------ | \------ | \------ || **wazuh-manager** | Ubuntu Server 24.04 | 192.168.153.129 | Backend SIEM et Dashboard || **kessel-server** | Ubuntu Desktop 24.04 | 192.168.153.130 | Agent Linux \+ Docker \+ Suricata IDS || **windows-desktop** | Windows 10 Education | 192.168.153.131 | Agent Windows (poste employé) || **VMware NAT** | — | 192.168.153.1 | Routeur et passerelle par défaut |

##### Composants additionnels de sécurité

* **Docker & Nginx :**  Déploiement d'applications web conteneurisées pour simuler des services de production.  
* **Suricata IDS/IPS :**  Intégré pour la détection d'intrusions réseau, analysant les flux via le fichier eve.json avec plus de 50 000 règles actives.

#### 2\. Stratégies de Collecte des Logs

Wazuh offre deux approches complémentaires pour assurer une couverture exhaustive de l'infrastructure.

##### 2.1 Collecte via Agents

Utilisée pour les serveurs et postes de travail (Linux, Windows).

* **Installation :**  Via script officiel ou gestionnaire de paquets (apt, PowerShell).  
* **Configuration :**  L'agent est configuré pour pointer vers l'IP du manager dans le fichier ossec.conf.

##### 2.2 Collecte sans agent (Syslog)

Indispensable pour les équipements ne supportant pas l'installation d'agents (firewalls, routeurs, conteneurs restreints).

* **Configuration du Manager :**  Modification du ossec.conf pour accepter les flux sur le port 514/UDP.  
* **Exemple pfSense :**  Activation du renvoi des messages de logs vers le serveur distant dans les paramètres système.  
* **Exemple Docker :**  Utilisation de rsyslog configuré pour envoyer les logs .info vers l'IP du manager.

#### 3\. Analyse et Structuration des Données

Le passage d'un log brut à une alerte de sécurité suit un processus rigoureux.

##### Les Décodeurs (Wazuh Decoders)

Leur rôle est d'extraire et de normaliser les données. Un log SSH brut est ainsi décomposé en champs structurés : user, srcip, status.

* **Importance :**  Sans décodeur, aucune règle de détection ne peut s'appliquer.  
* **Ressources :**  Wazuh intègre des centaines de décodeurs natifs (Apache, Windows, Suricata, etc.).

##### Les Règles (Wazuh Rules)

Elles analysent les champs extraits par les décodeurs selon des conditions spécifiques.

* **Niveaux d'alerte :**  Échelle de 0 (informationnel) à 15 (attaque critique).  
* **Exemple de détection :**  Une règle peut être configurée pour détecter spécifiquement un échec de connexion SSH sur le compte root.

#### 4\. Langages de Requête et Investigation

Pour interroger les données collectées, deux langages distincts sont utilisés selon le contexte :

##### DQL (Dashboards Query Language)

Utilisé pour la recherche directe dans les logs au sein d'OpenSearch.

* **Syntaxe :**  champ:valeur (ex: user.name:admin).  
* **Capacités :**  Opérateurs booléens (AND, OR, NOT), wildcards (err\*), et requêtes de plage (\>, \<, \>=).

##### WQL (Wazuh Query Language)

Utilisé spécifiquement dans l'interface Wazuh pour filtrer les données internes.

* **Usages :**  Filtrage des agents, des vulnérabilités, de l'inventaire ou du SCA.  
* **Syntaxe :**  Opérateurs \=, \!=, \~ (contient). Exemple : agent.name \= webserver-1.

#### 5\. Preuves Opérationnelles et Alertes Détectées

Les tests d'intrusion et de configuration confirment l'efficacité du système.

##### Événements de sécurité identifiés

Le dashboard rapporte des alertes variées, confirmant la visibilité sur différents vecteurs :| Source | Description | Niveau | Rule ID || \------ | \------ | \------ | \------ || kessel-server | **BlackSun**  (Suspicious User Agent) | 7 | 86601 || kessel-server | Host-based anomaly detection (rootcheck) | 7 | 510 || kessel-server | Successful sudo to ROOT executed | 3 | 5402 || wazuh-manager | New dpkg (Debian Package) installed | 7 | 2902 |

##### Validation par simulation

* **Test IDS :**  L'utilisation de la commande curl \-A "BlackSun" http://testmynids.org/uid/index.html déclenche immédiatement une alerte Suricata de priorité 1 ("Network Trojan detected") visible dans Wazuh.  
* **Test Réseau :**  Un scan nmap sur le réseau pfSense génère des logs interceptés et centralisés par le manager via Syslog.

