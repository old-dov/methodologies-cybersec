# Analyse de Protocoles Réseau via Wireshark : Traceroute et Session BGP

## 1. Objectif

Deux exercices d'analyse de capture réseau (tcpdump/Wireshark) portant sur des mécanismes fondamentaux de l'acheminement Internet : la découverte de route via **traceroute** (TTL/ICMP), et l'établissement d'une session de routage **BGP** (TCP/QoS). Regroupés, ils couvrent les deux extrémités du sujet — comment un paquet trouve son chemin, et comment les routeurs négocient ce chemin entre eux.

## 2. Mécanique du traceroute (TTL et ICMP)

### Capture

```bash
sudo tcpdump -i eth0 udp or icmp -w traceroute_capture.pcap
traceroute 8.8.8.8
```

### Filtres d'analyse Wireshark

| Filtre | Usage |
|---|---|
| `udp && ip.dst == 8.8.8.8` | Isoler les sondes UDP envoyées vers la destination |
| `icmp.type == 11` | Réponses des routeurs intermédiaires (Time Exceeded) |
| `icmp.type == 3 && icmp.code == 3` | Réponse finale de la destination (Port Unreachable) |

### Le rôle réel du TTL

Le **TTL (Time-To-Live)** n'est pas une durée en secondes mais un **compteur de sauts**, dont la fonction primaire est d'empêcher un paquet de boucler indéfiniment sur Internet. Traceroute détourne ce mécanisme à des fins de diagnostic : le TTL est incrémenté de 1 à chaque nouvelle sonde, forçant chaque routeur successif du chemin à rejeter le paquet arrivé à expiration et à s'identifier via un message d'erreur ICMP.

### Pourquoi la réponse finale diffère

- Les routeurs **intermédiaires** répondent par **ICMP Type 11** (Time Exceeded) car le TTL expire avant d'atteindre la destination — ils indiquent ainsi leur présence sur le chemin sans jamais avoir reçu de requête qui leur était réellement destinée.
- Le **serveur final** reçoit le paquet avec un TTL suffisant, mais ne possède aucun service actif sur les ports UDP élevés visés par traceroute. Il renvoie donc **ICMP Type 3, Code 3** (Port Unreachable), qui confirme paradoxalement que la destination a bien été atteinte.

Cette asymétrie de réponse (Type 11 en chemin, Type 3/3 à l'arrivée) est le signal que traceroute interprète pour distinguer un simple saut intermédiaire de la destination finale.

## 3. Anatomie d'une session BGP

### Couche transport

BGP s'appuie sur **TCP port 179** pour garantir une transmission fiable des messages de routage — contrairement aux protocoles IGP comme OSPF qui opèrent directement sur IP.

| Paramètre | Valeur observée |
|---|---|
| Protocole IP | 6 (TCP) |
| Port de destination | 179 |
| Flags TCP (messages OPEN) | PSH + ACK |

La combinaison **PSH + ACK** sur les messages OPEN indique que le routeur confirme la réception des segments précédents (ACK) tout en demandant une transmission immédiate des données BGP à la couche applicative (PSH), sans attendre de bufferiser davantage.

### Séquence d'établissement

1. **OPEN** : négociation des paramètres de session (numéro d'AS, BGP Identifier).
2. **KEEPALIVE** : validation de la connexion et maintien de l'état *Established*.
3. **UPDATE** : annonce des préfixes (NLRI — Network Layer Reachability Information) et des chemins d'AS (AS_PATH).

Aucune route n'est échangée avant que la session ne soit passée en état *Established* via OPEN puis KEEPALIVE — le protocole négocie systématiquement la relation avant le contenu.

### Qualité de service (QoS)

Le marquage QoS repose sur le champ **DSCP (Differentiated Services Field)** de l'en-tête IP :

| Valeur DSCP | Signification | Observation |
|---|---|---|
| `0x00` (CS0) | Best Effort | Trafic standard, sans priorisation |
| `0xc0` (CS6) | Network Control | Priorité élevée réservée au trafic de contrôle réseau (BGP), pour éviter que ces messages ne soient supprimés en cas de congestion |

**Point d'attention pratique** : la présence de la classe CS6 dans une spécification ne garantit pas son application systématique sur chaque paquet capturé — la capture réelle peut révéler un marquage incohérent ou absent, signe d'une configuration QoS incomplète en amont (souvent sur les équipements intermédiaires plutôt que sur les routeurs BGP eux-mêmes).

## 4. Ce que ces deux analyses ont en commun

Traceroute et BGP illustrent chacun à leur niveau un principe identique : l'acheminement Internet repose sur des mécanismes de **négociation et de signalisation explicites**, jamais sur une simple transmission aveugle. Traceroute exploite un effet de bord du TTL pour cartographier un chemin ; BGP négocie formellement l'échange de routes avant tout transfert de données de routage. Dans les deux cas, l'analyse Wireshark révèle une mécanique de contrôle sous-jacente invisible à l'utilisateur final.

## 5. Conclusion

La lecture combinée de ces deux captures donne une vision cohérente de l'acheminement réseau à deux échelles : le chemin emprunté paquet par paquet (traceroute, TTL/ICMP) et la construction même de ce chemin au niveau du plan de contrôle inter-AS (BGP, TCP/UPDATE). Comprendre l'un éclaire l'autre — le chemin observé par traceroute est la conséquence directe des routes que BGP a négociées en amont.
