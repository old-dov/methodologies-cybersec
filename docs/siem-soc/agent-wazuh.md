**M É T H O D O L O G I E P R O F E S S I O N N E L L E**
# **Installation de l'agent Wazuh**

Procédure de déploiement multi-systèmes — Windows · Linux · macOS


Guide unifié pour installer, enregistrer et vérifier un agent Wazuh sur un
poste ou un serveur, quel que soit son système d'exploitation, et l'intégrer
à une infrastructure Wazuh existante.


Windows 10 / 11 / Server Debian & Ubuntu RHEL / CentOS / Rocky / Alma


macOS (Intel & Apple Silicon)



**Version cible**

Wazuh Agent 4.14.x



**Compatibilité Manager**

≥ version de l'agent



**Ports requis**

1514/TCP · 1515/TCP



**Public**

Client / utilisateur final



Méthodologie professionnelle Agent Wazuh multi OS Page 1 / 7


## **1. Introduction**

Ce document présente la procédure complète pour installer et configurer l'agent Wazuh sur les principaux

systèmes d'exploitation : **Windows**, **Linux** (familles Debian/Ubuntu et RHEL/CentOS) et **macOS** . Il est destiné

à un client ou à un utilisateur final souhaitant intégrer un poste ou un serveur dans une infrastructure Wazuh

existante.

Chaque section OS est autonome : sélectionnez celle qui correspond à la machine à superviser. Les étapes de

vérification, de test et de dépannage en fin de document s'appliquent à tous les systèmes.


**Règle de compatibilité.** La version de l'agent doit toujours être inférieure ou égale à celle du Wazuh Manager. Un

agent plus récent que le Manager peut présenter un comportement instable. Vérifiez la version du Manager avant de
déployer, puis choisissez une version d'agent identique ou antérieure.

## **2. Pré-requis communs**











Droits **administrateur** (Windows) ou **root / sudo** (Linux, macOS) sur la machine.

Connectivité réseau vers le Wazuh Manager :








**1514/TCP** - remontée des événements (agent → manager).

**1515/TCP** - enrôlement / échange de clé ( `agent-auth` ) .



Adresse IP ou nom d'hôte du Manager disponible.

Le cas échéant, le mot de passe d'enregistrement fourni par l'administrateur Wazuh.






## **3. Deux méthodes de déploiement**

**Méthode** **Description & usage recommandé**






|A — Dashboard Wazuh<br>(recommandée)|Le Dashboard génère automatiquement la commande d'installation exacte, avec<br>la bonne version de paquet et les bons paramètres. Menu : Agents<br>management → Summary → Deploy new agent, puis sélection de l'OS/<br>architecture. Idéal pour un déploiement ponctuel ou par un utilisateur non<br>technique.|
|---|---|
|**B — Ligne de commande**<br>(déploiements de masse)|Installation via le gestionnaire de paquets ou l'installeur, avec les variables<br>d'enrôlement passées en une seule commande. Détaillée par OS dans les<br>sections 4 à 7. Idéale pour l'automatisation (script, Ansible, GPO/MSI<br>silencieux).|



**Conseil.** Utilisez le Dashboard (méthode A) pour obtenir la version de paquet à jour, puis reproduisez la commande

fournie ci-après en l'adaptant à votre parc.


Méthodologie professionnelle — Agent Wazuh multi-OS Page 2 / 7


**Systèmes :** Windows 10, Windows 11, Windows Server 2016 et versions ultérieures.


**4.1 Téléchargement**

Récupérer l'installeur depuis le Dashboard ( **Deploy new agent → Windows → 64 bits** ) ou directement le

paquet `wazuh-agent-4.14.x-1.msi` .


**4.2 Installation silencieuse (PowerShell administrateur)**

```
  # Depuis le dossier contenant le .msi, PowerShell en administrateur
  .\wazuh-agent-4.14.x-1.msi /q `
   WAZUH_MANAGER="IP_DU_MANAGER" `
   WAZUH_AGENT_NAME="poste-windows-01" `
   WAZUH_AGENT_GROUP="windows-workstations"

```

Variante `msiexec` (invite de commandes / GPO) :

```
  msiexec.exe /i wazuh-agent-4.14.x-1.msi /q WAZUH_MANAGER="IP_DU_MANAGER"
  WAZUH_AGENT_NAME="poste-windows-01"

```

L'agent s'installe dans `C:\Program Files (x86)\ossec-agent\` .


**4.3 Installation graphique (alternative)**




- Double-cliquer sur le fichier `.msi`, accepter les conditions, installation standard.

- Ouvrir **Wazuh Agent Manager**, renseigner l'adresse du Manager, puis **Save** .


**4.4 Démarrage du service**

```
NET START WazuhSvc
# Pour redémarrer après une modification de configuration :
NET STOP WazuhSvc
NET START WazuhSvc

```




**Systèmes :** Debian, Ubuntu, Linux Mint, Kali et dérivés APT. Commandes à exécuter en **root** (ou `sudo` ) .


**5.1 Ajout du dépôt Wazuh**

```
  apt-get install gnupg apt-transport-https

  curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | \
  gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import \

```

Méthodologie professionnelle — Agent Wazuh multi-OS Page 3 / 7


```
&& chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/
stable main" | \
tee -a /etc/apt/sources.list.d/wazuh.list

apt-get update

```

**5.2 Installation et enrôlement**

```
WAZUH_MANAGER="IP_DU_MANAGER" \
WAZUH_AGENT_NAME="serveur-ubuntu-01" \
WAZUH_AGENT_GROUP="linux-servers" \
apt-get install wazuh-agent

```

**5.3 Activation du service**

```
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent

```




**Systèmes :** RHEL, CentOS, Rocky Linux, AlmaLinux, Fedora et dérivés RPM. Commandes en **root** .


**6.1 Ajout du dépôt Wazuh**

```
  rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH

  cat > /etc/yum.repos.d/wazuh.repo << EOF
  [wazuh]
  gpgcheck=1
  gpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH
  enabled=1
  name=EL-\$releasever - Wazuh
  baseurl=https://packages.wazuh.com/4.x/yum/
  protect=1
  EOF

```

**6.2 Installation et enrôlement**

```
  WAZUH_MANAGER="IP_DU_MANAGER" \
  WAZUH_AGENT_NAME="serveur-rocky-01" \
  WAZUH_AGENT_GROUP="linux-servers" \
  yum install wazuh-agent

  # Sur les distributions récentes, remplacez "yum" par "dnf" si besoin.

```

Méthodologie professionnelle — Agent Wazuh multi-OS Page 4 / 7


**6.3 Activation du service**

```
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent

```




**Systèmes :** macOS Sierra et ultérieurs (Intel) ; macOS Big Sur et ultérieurs (Apple Silicon). Commandes en

**sudo** .


**7.1 Choix du paquet selon l'architecture**


**Architecture** **Paquet**

|Intel (x86_64)|wazuh-agent-4.14.x-1.intel64.pkg|
|---|---|
|Apple Silicon (ARM64)|`wazuh-agent-4.14.x-1.arm64.pkg`|



**7.2 Téléchargement, installation et enrôlement**

```
  # Exemple pour Apple Silicon — adapter le nom de paquet pour Intel
  curl -O https://packages.wazuh.com/4.x/macos/wazuh-agent-4.14.x-1.arm64.pkg

  echo "WAZUH_MANAGER='IP_DU_MANAGER' WAZUH_AGENT_NAME='mac-01' WAZUH_AGENT_GROUP='macos  workstations'" \
  > /tmp/wazuh_envs

  sudo installer -pkg wazuh-agent-4.14.x-1.arm64.pkg -target /

```

**7.3 Démarrage du service**

```
  sudo launchctl bootstrap system /Library/LaunchDaemons/com.wazuh.agent.plist

## **8. Enrôlement manuel (méthode alternative, tous OS)**

```

Si l'agent a été installé sans variables d'enrôlement, ou pour ré-enregistrer un agent, utilisez l'utilitaire `agent-`

`auth` qui échange la clé avec le Manager sur le port **1515** .


**OS** **Commande**

|Windows|& "C:\Program Files (x86)\ossec-agent\agent-auth.exe" -m<br>IP_DU_MANAGER -p 1515|
|---|---|
|Linux / macOS|`/var/ossec/bin/agent-auth -m IP_DU_MANAGER -p 1515`|



Méthodologie professionnelle — Agent Wazuh multi-OS Page 5 / 7


## **9. Vérification de la connexion**

Côté **agent**, contrôler la version et l'état :


**OS** **Commande**

|Windows|& "C:\Program Files (x86)\ossec-agent\wazuh-control.exe" status|
|---|---|
|Linux / macOS|`/var/ossec/bin/wazuh-control info | grep version`|



Côté **Dashboard** : **Agents management → Summary** . Le nouvel agent doit apparaître avec le statut **Active** .

## **10. Tests fonctionnels**


**10.1 Générer un événement**


**OS** **Action de test**

|Windows|PowerShell : Get-EventLog -LogName Security -Newest 1 (ou une ouverture de<br>session).|
|---|---|
|Linux|Provoquer un événement d'authentifcation, ex. `sudo -k; sudo whoami`, ou une<br>connexion SSH.|
|macOS|Une authentifcation `sudo` ou une ouverture de session utilisateur.|



**10.2 Vérifier la remontée dans Wazuh**

Dans le Dashboard : **Threat Hunting / Security Events**, filtrer par agent. Les événements doivent apparaître

quasiment en temps réel.

## **11. Dépannage**


**Symptôme** **Actions**







|Agent non connecté (Never<br>connected / Disconnected)|Vérifier l'ouverture des ports 1514 et 1515 vers le Manager ; contrôler l'IP du<br>Manager dans la configuration ; vérifier la clé d'enregistrement ; redémarrer le<br>service.|
|---|---|
|Clé invalide / enrôlement refusé|Régénérer une clé côté Manager (**Agents → Manage agents / keys**), puis ré-<br>enrôler l'agent via `agent-auth` .|
|Pare-feu Windows|Autoriser `agent-auth.exe` et `wazuh-agent.exe`, ainsi que les ports 1514<br>et 1515.|
|Diagnostic par journaux||


Méthodologie professionnelle — Agent Wazuh multi-OS Page 6 / 7


## **12. Désinstallation (référence)**

|Windows|msiexec.exe /x wazuh-agent-4.14.x-1.msi /q|
|---|---|
|Debian / Ubuntu|`apt-get remove --purge wazuh-agent`|
|RHEL / CentOS|`yum remove wazuh-agent`|
|macOS|Script `/Library/Ossec/uninstall.sh` (fourni par le paquet).|


## **13. Conclusion**

L'agent Wazuh est désormais installé, enrôlé et opérationnel sur le système cible, quel que soit son OS. La

machine remonte ses événements au Manager et peut être supervisée selon les politiques de sécurité définies.

Pour un parc hétérogène, privilégiez la génération de commande via le Dashboard afin de garantir la

cohérence des versions entre agents et Manager.


Méthodologie professionnelle — Agent Wazuh multi-OS Page 7 / 7


