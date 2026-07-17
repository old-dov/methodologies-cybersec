# Guide Fondamental : La Sainte Trinité du Réseau Windows (AD, DNS, DHCP)

#### 1\. Introduction : L'Écosystème Invisible

Bienvenue dans les coulisses de l'infrastructure Microsoft. Pour un administrateur système, comprendre comment une machine communique ne suffit pas ; il faut maîtriser l'intelligence qui orchestre ces échanges. Imaginez le réseau comme une ville bien organisée où trois services essentiels travaillent main dans la main pour que chaque citoyen (utilisateur) puisse accéder à son bureau :

* **Active Directory (AD) :**  Le cerveau et le gardien. Il gère l'identité (qui êtes-vous ?) et les permissions (qu'avez-vous le droit de faire ?).  
* **DNS :**  La boussole. Il traduit les noms humains (ex: internal.stellar.local) en coordonnées GPS techniques (adresses IP).  
* **DHCP :**  Le distributeur automatique. Il attribue dynamiquement une configuration réseau à chaque nouvel arrivant pour qu'il puisse communiquer immédiatement.Mais attention : avant de pouvoir distribuer des instructions au reste du monde, le serveur lui-même doit être solidement ancré au sol.

#### 2\. La Fondation : Pourquoi l'IP Statique est Non-Négociable

Imaginez que la mairie de votre ville change d'adresse postale tous les matins sans prévenir. Le courrier n'arriverait jamais. C'est exactement ce qui se passe si votre serveur (comme  **DC1**  ou  **STORAGE**  dans notre environnement StellarTech) change d'adresse IP. Un serveur de rôle doit être un point de repère fixe et fiable.Avant d'installer le moindre rôle (DHCP, DNS ou AD), vous devez impérativement configurer une  **adresse IP statique**  sur votre serveur. Si le serveur change d'adresse, les clients perdront leur "boussole" (DNS) et leur "distributeur" (DHCP), entraînant une paralysie totale du domaine.Une fois que notre serveur possède une identité fixe, il peut commencer à distribuer les paramètres réseau aux autres membres de la famille : les clients.

#### 3\. Le DHCP : Le Distributeur Automatique de Paramètres

Le protocole DHCP (Dynamic Host Configuration Protocol) est votre meilleur allié pour éviter la gestion manuelle fastidieuse. Son outil principal est l' **Étendue (Scope)**  : une plage d'adresses que le serveur est autorisé à "prêter" aux machines.| Composant | Utilité pour le client || \------ | \------ || **Plage d'IP** | L'adresse unique attribuée à la machine pour la durée de son bail. || **Masque de sous-réseau** | Délimite la frontière entre le réseau local et le reste du monde. || **Passerelle (Gateway)** | L'adresse du routeur pour sortir du réseau local. || **Serveur DNS** | L'adresse indispensable pour résoudre les noms de domaine. |  
**Le conseil de l'expert :**  En production, on évite d'installer le DHCP directement sur un Contrôleur de Domaine (DC) pour des raisons de sécurité et de performance. Cependant, dans notre environnement de laboratoire, il est acceptable d'utiliser  **DC1**  pour ce rôle.**Sécurité (Authorization) :**  Pour éviter qu'un "Rogue DHCP" (un serveur pirate) ne s'installe sur votre réseau, Windows exige que votre serveur DHCP soit  **autorisé**  dans l'Active Directory via la commande PowerShell : Add-DhcpServerInDC. Sans cette étape, le service refusera de distribuer la moindre IP.

#### 4\. Le DNS : La Boussole et les Enregistrements Magiques (SRV)

Sans DNS, l'Active Directory est aveugle. Le DNS ne se contente pas de traduire des sites web ; il permet aux machines de localiser les services de sécurité indispensables.

##### Les enregistrements SRV : Les panneaux indicateurs

Active Directory crée automatiquement des enregistrements de service (SRV) essentiels. Sans eux, une machine peut avoir une IP, mais elle ne saura jamais à qui s'adresser pour se connecter.Le format standard est : \_service.\_protocol.domainePour trouver l'annuaire (LDAP) : \_ldap.\_tcp.stellar.localPour l'authentification sécurisée : \_kerberos.\_tcp.stellar.local

##### L'exemple concret de StellarTech

Dans notre infrastructure, nous avons créé un  **enregistrement de type A**  sur le DNS de  **DC1** . Cet enregistrement permet de pointer le nom internal.stellar.local vers l'adresse IP du serveur  **STORAGE**  (192.168.122.13), où se trouve notre portail intranet.Voici comment vérifier que votre boussole fonctionne avec l'outil nslookup (exécuté ici depuis un client) :  
C:\\\> nslookup internal.stellar.local  
Serveur :  dc1.stellar.local  
Address:   192.168.122.10

Nom :      internal.stellar.local  
Address:   192.168.122.13

#### 5\. L'Interdépendance : Le Cycle de Vie d'une Connexion

Comprendre ces services individuellement est une chose, mais leur force réside dans leur collaboration. Suivons le voyage d'une donnée lorsqu'un utilisateur, comme  **Anakin** , allume son poste de travail :

1. **Attribution (DHCP) :**  Le poste d'Anakin demande une configuration. Le DHCP lui répond :  *"Voici ton IP, et voici l'adresse de ton serveur DNS."*  
2. **Localisation (DNS) :**  Le poste veut rejoindre le domaine stellar.local. Il interroge le DNS :  *"Où sont les enregistrements SRV*  *\_ldap*  *et*  *\_kerberos*  *?"*  Le DNS répond :  *"Ils sont sur DC1 \!"*  
3. **Authentification (AD) :**  Le poste contacte  **DC1** . L'Active Directory vérifie les identifiants d'Anakin et, si tout est correct, autorise l'ouverture de session et l'accès aux ressources.Ce cycle est le cœur de la stabilité d'un réseau Windows.

#### 6\. Cas Pratique : L'Infrastructure StellarTech

Voici comment les rôles sont répartis dans l'environnement que nous avons déployé. Notez l'utilisation d'un  **RODC**  (Read-Only Domain Controller) pour la redondance sécurisée.| Ressource | Rôle Principal | Adresse / Nom || \------ | \------ | \------ || **DC1** | Contrôleur de Domaine (PDC) & DNS Maître | 192.168.122.10 / dc1.stellar.local || **DC2** | Contrôleur Secondaire (RODC) | 192.168.122.11 / dc2.stellar.local || **STORAGE** | Serveur Web (IIS) & VPN (RRAS) | 192.168.122.13 / internal.stellar.local |

#### 7\. Conclusion et Check-list de l'Administrateur

Félicitations \! Vous avez maintenant une vision claire de la "Sainte Trinité" du réseau. Un bon administrateur ne devine pas, il vérifie. Avant de déclarer votre infrastructure opérationnelle, passez toujours par cette check-list :

*   **IP Statique :**  Tous mes serveurs (DC1, DC2, STORAGE) ont-ils des adresses fixes configurées ?  
*   **Autorisation DHCP :**  Le serveur a-t-il été autorisé dans l'AD (Add-DhcpServerInDC) ?  
*   **Scope DHCP :**  L'étendue est-elle activée et la plage d'IP est-elle suffisante ?  
*   **Enregistrements SRV :**  La commande nslookup \-type=srv \_ldap.\_tcp.stellar.local renvoie-t-elle bien l'adresse de mon DC ?  
*   **Résolution Type A :**  Le nom internal.stellar.local pointe-t-il bien vers 192.168.122.13 ?  
*   **Redondance :**  Mon RODC (DC2) est-il prêt à prendre le relais pour l'authentification en cas de coupure de DC1 ?Continuez à pratiquer, c'est ainsi que l'on forge une expertise solide \!

