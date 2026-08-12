# Protocole d'Audit Active Directory : Empoisonnement LLMNR & Relais NTLM

## 1. Synthèse exécutive

Cet audit couvre deux vulnérabilités chaînées liées aux protocoles d'authentification hérités de Windows, exploitées avec succès sur un domaine type (`TwinStar`) :

1. **L'empoisonnement LLMNR/NBT-NS**, qui permet d'intercepter des hachages de mots de passe réseau.
2. **Le relais NTLM (NTLM Relay)**, qui va plus loin en contournant totalement l'authentification pour usurper l'identité d'un compte, sans jamais avoir besoin de casser le hash intercepté.

L'absence de durcissement de base (résolution de noms héritée active, signature SMB non imposée) laisse la configuration par défaut de Windows vulnérable à une compromission complète du domaine.

## 2. Vulnérabilité 1 — Empoisonnement LLMNR & NBT-NS

**Mécanisme** : lorsqu'une station cherche à joindre une ressource réseau inexistante (faute de frappe, partage supprimé), la résolution DNS échoue. Windows bascule alors par défaut sur les protocoles de secours **LLMNR (Link-Local Multicast Name Resolution)** et **NBT-NS (NetBIOS Name Service)**, qui envoient une requête de diffusion (broadcast) sur le réseau local — sans authentifier la source de la réponse.

**Exploitation** : un outil d'interception comme **Responder** (v3.1.5.0) écoute ces diffusions et répond frauduleusement à la place de la ressource recherchée. Croyant s'adresser au bon serveur, la machine victime envoie son empreinte d'authentification, le **hash Net-NTLMv2**.

**Impact** : le mot de passe n'est pas transmis en clair, mais le hash capturé peut être soumis hors-ligne à une attaque par dictionnaire ou brute-force (John the Ripper, Hashcat) pour être retrouvé en clair.

## 3. Vulnérabilité 2 — Relais NTLM (NTLM Relay)

**Mécanisme** : au lieu de tenter de casser le hash Net-NTLMv2 capturé (ce qui peut échouer si le mot de passe est complexe), l'attaquant se positionne en intermédiaire actif (Man-in-the-Middle) et **rejoue immédiatement** l'authentification interceptée vers une autre cible du domaine.

**Exploitation** : les modules SMB et HTTP de Responder sont désactivés pour laisser le champ libre à `impacket-ntlmrelayx`. Lorsqu'une machine émet une requête de connexion erronée, l'authentification NTLM est interceptée puis relayée vers une cible valide (ex. `SRV2`).

```
# Poisoning LLMNR/NBT-NS (modules SMB/HTTP désactivés dans Responder.conf)
responder -I eth0

# Relais vers une cible du domaine
impacket-ntlmrelayx -tf targets.txt -smb2support
```

**Impact** : la cible relayée accepte l'authentification sans vérifier la légitimité du serveur d'origine. Si le compte relayé possède des privilèges élevés (Administrateur du domaine), l'attaquant obtient un accès complet instantané (shell interactif, `smbexec.py`) **sans jamais connaître ni casser le mot de passe**.

## 4. Détection

| Indicateur | Description |
|---|---|
| Trafic broadcast anormal | Pic de requêtes LLMNR/NBT-NS inhabituel sur le segment réseau |
| Échecs de résolution DNS répétés | Précèdent souvent une tentative de poisoning |
| Authentifications SMB croisées | Un compte s'authentifiant depuis une IP inhabituelle vers plusieurs hôtes en séquence rapide, signature d'un relais actif |

## 5. Plan de remédiation et durcissement

Contre-mesures à appliquer par GPO au niveau du domaine.

### A. Désactiver la résolution de noms héritée

- **LLMNR** : `Configuration ordinateur > Modèles d'administration > Réseau > Client DNS` → activer *Désactiver la résolution de noms multidiffusion*.
- **NetBIOS (NBT-NS)** : à désactiver sur les propriétés de la carte réseau (`TCP/IPv4 > Avancé > onglet WINS > Désactiver NetBIOS sur TCP/IP`) ou via les options du serveur DHCP.

### B. Imposer la signature SMB (parade absolue contre le NTLM Relay)

Si la communication est signée numériquement, toute tentative de relais invalide la signature et la session est rejetée par la cible.

- `Configuration ordinateur > Paramètres Windows > Paramètres de sécurité > Stratégies locales > Options de sécurité`
- Activer : *Serveur réseau Microsoft : signer numériquement les communications (toujours)*
- Activer : *Client réseau Microsoft : signer numériquement les communications (toujours)*

### C. Protections complémentaires

- **Extended Protection for Authentication (EPA)** sur les serveurs IIS et les services AD CS, pour empêcher le relais NTLM vers des interfaces HTTP/HTTPS.
- **Transition vers Kerberos** : restreindre au maximum l'usage de NTLM au sein du domaine, Kerberos n'étant pas sujet à ce type de relais.

## 6. Conclusion

LLMNR Poisoning et NTLM Relay illustrent un même défaut de fond : des protocoles hérités sans authentification mutuelle, encore actifs par défaut. La désactivation de la résolution de noms héritée et l'imposition de la signature SMB suffisent à neutraliser la quasi-totalité de la chaîne d'attaque, sans dépendre uniquement de la détection.
