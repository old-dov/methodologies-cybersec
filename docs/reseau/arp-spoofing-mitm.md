# Attaque Man-in-the-Middle par ARP Spoofing

## 1. Principe

Le protocole **ARP (Address Resolution Protocol)** ne vérifie par conception ni l'authenticité ni l'origine des réponses qu'il reçoit : toute machine du réseau local peut répondre à une requête ARP en se faisant passer pour une autre. L'attaque **ARP Spoofing** (ou empoisonnement ARP) exploite directement cette absence de vérification pour s'interposer entre deux machines et intercepter leur trafic — une attaque de type **Man-in-the-Middle (MitM)**.

## 2. Méthodologie de l'attaque

L'attaque se déroule en quatre phases sur une machine attaquante (Kali/Debian) positionnée sur le même segment réseau que les deux victimes.

### A. Préparation

Installation des outils nécessaires (`iproute2`, `dsniff` pour `arpspoof`).

### B. Activation du routage IP (condition indispensable)

Sans cette étape, l'interception casse la connectivité des victimes et l'attaque est immédiatement détectée (perte de connexion). Pour rester **transparente**, l'attaquant doit continuer à relayer le trafic après l'avoir intercepté :

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

### C. Empoisonnement bidirectionnel

```bash
# Faire croire à VPC1 que l'attaquant est VPC2
arpspoof -i eth0 -t <IP_VPC1> <IP_VPC2>

# Faire croire à VPC2 que l'attaquant est VPC1
arpspoof -i eth0 -t <IP_VPC2> <IP_VPC1>
```

Chaque victime met à jour sa table ARP locale, associant désormais l'adresse IP du correspondant légitime à l'adresse **MAC de l'attaquant**.

### D. Validation

Comparaison des tables ARP avant/après (`arp -a` ou `show arp`) : les adresses MAC des correspondants ont été remplacées par celle de la machine attaquante des deux côtés, confirmant l'interposition complète du trafic.

## 3. Pourquoi l'IP forwarding est le point critique

C'est l'étape la plus souvent sous-estimée : sans `ip_forward` activé, l'attaquant absorbe le trafic mais ne le retransmet pas, ce qui coupe la communication entre les victimes de façon immédiatement visible. Une attaque ARP Spoofing correctement menée est **invisible du point de vue applicatif** — les utilisateurs continuent de naviguer normalement pendant que leur trafic est intercepté.

## 4. Détection

| Indicateur | Description |
|---|---|
| **Table ARP incohérente** | Une même adresse MAC associée à plusieurs adresses IP sur un hôte |
| **Trames ARP gratuites en excès** | Volume anormal de réponses ARP non sollicitées (`gratuitous ARP`) |
| **Latence ajoutée** | Un saut supplémentaire (l'attaquant) peut introduire une latence mesurable, bien que souvent négligeable sur un LAN |
| **Outils dédiés** | `arpwatch` (surveillance des associations IP/MAC dans le temps), alertes IDS sur pattern de flooding ARP |

## 5. Contre-mesures

- **Dynamic ARP Inspection (DAI)** : sur les commutateurs manageables, valide chaque paquet ARP contre une table de correspondance IP/MAC de confiance (généralement construite via DHCP Snooping), et bloque les réponses ARP illégitimes au niveau du port.
- **Port Security** : limite le nombre d'adresses MAC autorisées par port et bloque les changements d'adresse MAC suspects.
- **ARP statique** : sur les liaisons critiques (serveurs, passerelles), fixer manuellement les entrées ARP élimine le risque d'empoisonnement, au prix d'une perte de flexibilité opérationnelle.
- **Chiffrement de bout en bout** : ARP Spoofing n'expose que du trafic en clair. TLS/HTTPS ne bloque pas l'interception mais rend le contenu illisible pour l'attaquant.

## 6. Conclusion

ARP Spoofing rappelle qu'un protocole de couche 2 sans authentification reste exploitable même dans un réseau parfaitement segmenté au niveau IP. La défense n'est pas applicative mais **structurelle** : elle se joue sur la configuration des commutateurs (DAI, Port Security), pas sur le renforcement des hôtes eux-mêmes.
