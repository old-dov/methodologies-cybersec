# Structurer un Plan de Réponse à Incident (Cas Tyrell Corporation)

## 1. Objectif de la fiche

Un Post-Incident Report (voir [Post-Incident Report : Ransomware IBC Bank](post-incident-ransomware-ibc-bank.md)) documente ce qui s'est passé **pendant** un incident. Un **Incident Response Plan (IRP)** documente ce qui doit se passer **avant** qu'il n'arrive : qui décide, qui agit, dans quel délai, et par quel canal. Cette fiche synthétise la structure d'un IRP à partir du cas Tyrell Corporation (division recherche génétique, Barcelone), un document gouvernance versionné et revu annuellement.

Un IRP efficace répond à quatre questions à froid, pour ne plus avoir à les débattre à chaud : **Qui décide ? Comment classifie-t-on ? Qui a le droit de couper quoi ? Comment communique-t-on sans compromettre la réponse elle-même ?**

## 2. Rôles et responsabilités

L'équipe de réponse à incident (IRT) est activée par déclaration formelle de l'IR Manager.

| Rôle | Responsabilité |
|---|---|
| **IR Manager (CISO)** | Supervise le processus, déclare le niveau de sévérité, coordonne les parties prenantes, reporte à la direction |
| **Technical Lead** | Dirige l'investigation technique, coordonne l'isolation des systèmes, gère la restauration |
| **Équipe SOC** (astreinte 24/7) | Surveille SIEM/IDS, effectue le triage initial, escalade au Technical Lead sous 30 minutes après validation |
| **Communications Officer** | Gère les communications internes/externes pendant l'incident |

**Procédure d'activation** : à la déclaration d'un incident, tous les membres de l'IRT sont notifiés via un canal dédié. Accusé de réception attendu sous **15 minutes** en heures ouvrées, **60 minutes** hors heures ouvrées. Le Technical Lead et le SOC opèrent en astreinte continue.

## 3. Détection, classification et escalade

### Infrastructure de détection
- **IDS** (ex. Suricata) au périmètre réseau et entre segments critiques, signatures mises à jour régulièrement.
- **SIEM** (ex. Splunk) ingérant les logs AD, pare-feux, agents endpoint, VPN, serveurs applicatifs, avec règles de corrélation sur brute-force, mouvement latéral, exfiltration.

### Processus de triage (SOC)
1. Revue de l'alerte sous 15 minutes.
2. Vérification par rapport aux faux-positifs connus (base de connaissance SOC).
3. Si confirmée : collecte du contexte initial (systèmes affectés, horodatage, source, alertes corrélées sur 24h).
4. Escalade au Technical Lead avec résumé écrit.

### Catégories de classification

| Catégorie | Exemples |
|---|---|
| Infection par malware | Ransomware, trojan, ver |
| Accès non autorisé | Identifiants compromis, accès système confirmé |
| Violation de données | Exfiltration ou exposition confirmée/suspectée de données sensibles |
| Déni de service | Atteinte à la disponibilité |
| Menace interne | Action malveillante ou négligente d'un employé/prestataire |

Sur cette base, le Technical Lead informe l'IR Manager, qui décide du niveau de réponse et de l'activation complète ou partielle de l'IRT.

## 4. Procédures de confinement et d'isolation

### Isolation réseau
- **Réaffectation VLAN** : bascule des systèmes compromis vers un VLAN de quarantaine pré-configuré, sans connectivité vers la production.
- **Mise à jour des règles pare-feu** : blocage des IP/domaines malveillants connus au périmètre.
- **Arrêt de serveurs** si l'isolation réseau seule ne suffit pas à stopper les processus malveillants.

### Isolation des comptes et identifiants
1. Désactivation du/des compte(s) compromis dans l'AD.
2. Réinitialisation forcée du mot de passe des utilisateurs affectés (et des comptes à identifiants similaires).
3. Révocation des sessions et jetons actifs.
4. Revue des journaux d'authentification récents pour détecter un accès non autorisé additionnel.

### Matrice d'autorité de décision

| Action | Heures ouvrées | Hors heures ouvrées |
|---|---|---|
| Réaffectation VLAN, désactivation de compte | Technical Lead | SOC d'astreinte, de manière autonome |
| Arrêt de serveur | Technical Lead | Nécessite l'accord verbal du Technical Lead ou de l'IR Manager |

**Point clé** : formaliser à l'avance qui peut agir sans validation évite la paralysie décisionnelle au moment critique — c'est souvent ce délai de validation qui creuse l'écart entre détection et confinement effectif (voir le cas IBC Bank, où ce délai a été de plus de 4 heures).

## 5. Notification et communication

- **Communication interne** : canal dédié, **séparé** de la messagerie corporate standard, pour réduire le risque d'interception si l'infrastructure corporate est elle-même compromise. Mises à jour de statut au moins toutes les 2 heures pendant un incident actif.
- **Notification à la direction** : dès la déclaration formelle, puis mise à jour au minimum toutes les 4 heures (immédiatement si le périmètre ou la sévérité évolue significativement).
- **Notification réglementaire** : en cas de violation de données personnelles confirmée ou suspectée, notification aux autorités compétentes selon les délais réglementaires applicables (ex. 72h RGPD).
- **Communication post-incident** : rapport de synthèse pour la direction incluant timeline, actions menées, recommandations — c'est le pont direct vers le Post-Incident Report.

## 6. Ce qui fait la qualité d'un IRP

- **Versionné et daté**, avec historique de révision explicite (qui a changé quoi, et pourquoi).
- **Rôles nommés**, pas seulement des fonctions abstraites — avec un annuaire de contacts tenu à jour.
- **Autorité de décision explicite**, y compris pour le cas dégradé (hors heures ouvrées).
- **Canal de communication de secours** distinct de l'infrastructure principale, pour rester opérationnel même si celle-ci est compromise.
- **Revu à échéance fixe** (ex. annuelle), pas seulement après un incident réel.

## 7. Conclusion

Un IRP n'a de valeur que s'il élimine l'improvisation au moment où elle coûte le plus cher. Le comparer à un cas réel comme IBC Bank est instructif : chaque minute perdue dans le cas IBC Bank correspond à une question que l'IRP de Tyrell a, elle, déjà tranchée à froid — qui décide, qui agit, et jusqu'où sans validation.
