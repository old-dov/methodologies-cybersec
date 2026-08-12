# Post-Incident Report : Attaque Ransomware avec Exfiltration (Cas IBC Bank)

## 1. Synthèse exécutive

**Sévérité : CRITIQUE.** Les 2 et 3 mars 2025, IBC Bank subit une attaque combinant ransomware et exfiltration de données. L'attaque débute par un email de phishing ouvert par un membre de l'équipe IT (10h32), installant un cheval de Troie d'accès distant (RAT) sur le poste `WKS-0347`. À partir de ce point d'ancrage, les attaquants extraient les identifiants Domain Admin du contrôleur de domaine via **Mimikatz**, puis se déplacent latéralement en **RDP** vers les serveurs de transaction — un mouvement rendu possible par l'**absence de segmentation réseau** entre le VLAN utilisateurs et le VLAN production.

Le chiffrement débute à 03h02 sur `TXNSERV-01`, se propage à `TXNSERV-02` puis au serveur de sauvegarde `BKPSRV-01`. Avant chiffrement, **4,2 Go de données clients** (~35 000 enregistrements : noms, adresses, IBAN, historiques de transactions) sont exfiltrés vers un serveur C2 connu. L'équipe IR est activée à 07h15 et isole les systèmes affectés à 09h00 ; la banque en ligne est indisponible 6 heures. La rançon de 5 BTC n'a pas été payée. Pertes financières directes préliminaires : **plus de 3 M€**.

## 2. Timeline de l'incident

| Horodatage | Événement |
|---|---|
| 02/03 — 10h32 | Email de phishing ouvert par un employé IT ; macro malveillante exécutée ; RAT installé sur `WKS-0347` (règle de détection macro en mode « log only », non bloquante) |
| 02/03 — 11h00 | Reconnaissance interne depuis `WKS-0347` : identification des serveurs de transaction et du contrôleur de domaine |
| 03/03 — 03h00 | Exécution de **Mimikatz** sur `DC-01` : extraction des identifiants Domain Admin |
| 03/03 — 03h00 | Sessions RDP établies de `WKS-0347` vers `TXNSERV-01`/`02` avec les identifiants compromis |
| 03/03 — 03h02 | Début du chiffrement sur `TXNSERV-01` (extension `.ibclocked`) ; alerte IDS Suricata déclenchée |
| 03/03 — 03h05 | Début de l'exfiltration vers `185.247.226.129` en HTTPS — ~4,2 Go transférés en 40 min |
| 03/03 — 03h47 | Alerte SIEM de corrélation : transfert sortant volumineux vers une IP inconnue |
| 03/03 — 06h30 | Le ransomware se propage à `BKPSRV-01` (sauvegardes récentes chiffrées) |
| 03/03 — 07h15 | L'analyste SOC contacte l'IR Manager — activation formelle de la réponse à incident |
| 03/03 — 09h00 | Isolation réseau de `TXNSERV-01`, `TXNSERV-02`, `BKPSRV-01` ; équipe forensique externe engagée |
| 04-05/03 | Récupération à partir de sauvegardes antérieures à 48h ; réconciliation manuelle des transactions ; services rétablis après 6h d'interruption |
| 06/03 | Communication publique ; notification à l'autorité de protection des données (CNPD) dans le délai RGPD de 72h |

## 3. Cartographie MITRE ATT&CK

| Phase | Technique | ID MITRE | Détail |
|---|---|---|---|
| **Accès initial** | Spearphishing Attachment | T1566.001 | `.docx` piégé avec macro, envoyé à l'équipe IT → exécution PowerShell → RAT sur `WKS-0347` |
| **Élévation de privilèges** | OS Credential Dumping: LSASS Memory | T1003.001 | Mimikatz sur `DC-01` (pas de LSA Protection) |
| **Mouvement latéral** | Remote Services: RDP | T1021.001 | RDP de `WKS-0347` vers `TXNSERV-01/02`, permis par l'absence de segmentation VLAN |
| **Exfiltration** | Exfiltration Over C2 Channel | T1041 | 4,2 Go en HTTPS vers `185.247.226.129` (infrastructure C2 connue) |
| **Impact** | Data Encrypted for Impact | T1486 | Chiffrement `.ibclocked` sur `TXNSERV-01/02` puis `BKPSRV-01` |

## 4. Constats forensiques

- **Point d'entrée** : macro Office exécutant un script PowerShell ; règle de détection en mode « log only » (non bloquant) → compromission passée inaperçue pendant **~16h30**.
- **Compromission des identifiants** : traces de Mimikatz sur `DC-01`, absence de LSA Protection.
- **Mouvement latéral** : sessions RDP confirmées de `WKS-0347` (VLAN utilisateurs, `10.10.1.47`) vers `TXNSERV-01/02` (VLAN production, `10.10.2.10/11`) — aucune règle de pare-feu ni ACL ne restreignait le trafic RDP entre segments.
- **Compromission des sauvegardes** : `BKPSRV-01` situé sur le **même VLAN** que les serveurs de production, avec uniquement des sauvegardes en ligne et mutables → aucun point de restauration récent disponible.
- **Exfiltration** : 4,2 Go transférés vers `185.247.226.129:443`, IP associée à une infrastructure C2 de ransomware connue.

## 5. Efficacité de la réponse à incident

| Métrique | Valeur |
|---|---|
| Présence non détectée de l'attaquant | 16h30 (entre compromission initiale et première alerte SIEM) |
| Détection → escalade | 4h13 (entre l'alerte SIEM à 03h02 et l'activation formelle à 07h15) |
| Confinement | 6h (entre l'alerte et l'isolation effective à 09h00) |
| Indisponibilité banque en ligne | 6h |
| Notification régulatoire | Dans le délai RGPD de 72h |

Le délai de 4h13 entre la première alerte et l'escalade formelle constitue le principal goulet d'étranglement : l'analyste d'astreinte a attendu la revue complète des alertes corrélées avant d'escalader, laissant l'attaque progresser sans entrave pendant cette fenêtre.

## 6. Causes racines et recommandations priorisées

| # | Cause racine | Recommandation | Priorité |
|---|---|---|---|
| 1 | Absence de segmentation réseau entre VLAN utilisateurs et VLAN production | Microsegmentation stricte (Utilisateurs / Production / DMZ / Management) avec règles pare-feu bloquant le RDP direct poste → production | **Critique** |
| 2 | Sauvegardes en ligne, non immuables, colocalisées avec la production | Sauvegardes immuables et/ou hors-ligne (air-gap) avec tests de restauration réguliers | **Critique** |
| 3 | Absence de durcissement des identifiants sur le contrôleur de domaine | LSA Protection / Credential Guard sur tous les DC + modèle d'administration à paliers (tiered administration) | **Haute** |
| 4 | Politique de détection des macros trop permissive (« log only ») | Passer en mode « block », désactiver les macros par défaut pour les documents provenant d'Internet, renforcer le sandboxing email | **Haute** |
| 5 | Délai d'escalade trop long après la première alerte | Playbooks de corrélation SIEM automatisés avec seuils d'escalade clairs pour les alertes de sévérité élevée | **Moyenne** |

## 7. Indicateurs de compromission (IOC)

**Fichiers**
- Extension de chiffrement : `.ibclocked`
- Note de rançon : `READ_ME.txt` à la racine des systèmes affectés

**Réseau**

| IP | Port | Description |
|---|---|---|
| 185.247.226.129 | 443/HTTPS | Serveur C2 principal — destination de l'exfiltration |
| 46.101.142.159 | 443/HTTPS | Nœud C2 secondaire (configuration du RAT) |
| 5.61.37.252 | 443/HTTPS | Adresse C2 de repli trouvée dans le binaire |

**Email**
- Domaine expéditeur : `supplier-invoice.net`
- Sujets observés : *« Invoice Due »*, *« Payment Request »*
- Pièce jointe : `.docx` avec macro exécutant du PowerShell

**Processus**
- `powershell.exe -executionpolicy bypass -file encrypted.ps1`
- Artefacts Mimikatz en mémoire sur `DC-01`
- Sessions RDP non autorisées du VLAN utilisateurs vers le VLAN production avec des identifiants Domain Admin

## 8. Conclusion

Ce cas illustre comment quatre défaillances structurelles simples — segmentation réseau absente, sauvegardes non immuables, durcissement des identifiants insuffisant, et politique macro permissive — transforment un phishing isolé en compromission bancaire totale. Aucune de ces causes racines n'est une vulnérabilité logicielle : ce sont des choix d'architecture et de configuration, ce qui signifie qu'elles sont pleinement sous le contrôle de l'organisation et évitables par du durcissement standard plutôt que par un correctif ponctuel.
