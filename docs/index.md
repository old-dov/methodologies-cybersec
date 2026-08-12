# Wiki Méthodologies Cybersécurité

Base de connaissances personnelle regroupant les méthodologies d'audit, de tests offensifs et d'architecture sécurité.

## Dernières mises à jour

- 2026-08-12 : Nouvelle catégorie **Active Directory** - [Kerberoasting](active-directory/kerberoasting.md), [Empoisonnement LLMNR & Relais NTLM](active-directory/llmnr-ntlm-relay.md), [Pass-the-Hash (PtH)](active-directory/pass-the-hash.md).
- 2026-08-12 : Nouvelle catégorie **Pentest** - [Audit Combiné Red/Blue Team (Cas Cylentra Robotics)](pentest/cylentra-red-blue-team.md).

??? note "Historique complet des mises à jour"
    - 2026-08-04 : Nouvelle fiche publiée - [Analyse Statique avec Ghidra](bibliotheque/ghidra-analyse-statique.md).
    - 2026-08-03 : Nouvelle fiche publiée - [DFIR Compromission WordPress (Cas Blue Sun Corporation)](bibliotheque/dfir-compromission-wordpress-blue-sun.md).
    - 2026-07-31 : Nouvelle fiche publiée - [Forensics Mémoire avec Volatility (Cas Weyland-Yutani)](bibliotheque/forensics-memoire-volatility-weyland-yutani.md).
    - 2026-07-31 : Nouvelle fiche publiée - [Investigation Réseau via Wireshark (Cas BookWorld)](bibliotheque/investigation-reseau-wireshark-bookworld.md).
    - 2026-07-30 : Nouveau rapport publié - [Audit IRP & Gestion de Crise Cybersécurité (Tyrell Corporation / IBC Bank)](rapport-methodologique-audit-irp-gestion-crise.md).
    - 2026-07-29 : Nouveau rapport publie - [Audit de Conformite RGPD (DocWagon Connect)](rapport-methodologique-audit-conformite-rgpd.md).
    - 2026-07-28 : Nouveau rapport publié - [Conformité RGPD & AIPD (GenomePath)](rapport-conformite-rgpd-aipd-genomepath.md).
    - 2026-07-28 : Nouveau rapport publié - [Audit RGPD Rocicorp Pharma](rapport-audit-rgpd-rocicorp-pharma.md).
    - 2026-07-24 : Nouveau rapport publié - [AWSGoat Module 2 – Déploiement LocalStack](rapport-awsgoat-module2-deploiement-localstack.md).
    - 2026-07-23 : Nouveau rapport publié - [IAM Vulnerable LocalStack](rapport-iam-vulnerable-localstack.md).
    - 2026-07-22 : Nouveau rapport publié - [AWSGoat Pentest](rapport-awsgoat-pentest.md).
    - 2026-07-20 : Ajout de la méthodologie Pentest Grey Box API REST IBC-News.
    - 2026-07-17 : Ajout du protocole d'audit sur l'escalade de privilèges via les services Windows.
    - 2026-07-17 : Ajout de la méthodologie Lab Decima / Samaritan OS (Contrôle d'accès multi-vecteurs).
    - 2026-07-17 : Publication de 10 nouvelles fiches dans la bibliothèque.

## Rapports

- [Audit IRP & Gestion de Crise Cybersécurité (Tyrell Corporation / IBC Bank)](rapport-methodologique-audit-irp-gestion-crise.md)
- [Audit de Conformite RGPD (DocWagon Connect)](rapport-methodologique-audit-conformite-rgpd.md)
- [AWSGoat Pentest](rapport-awsgoat-pentest.md)
- [IAM Vulnerable LocalStack](rapport-iam-vulnerable-localstack.md)
- [AWSGoat Module 2 – Déploiement LocalStack](rapport-awsgoat-module2-deploiement-localstack.md)
- [Conformité RGPD & AIPD (GenomePath)](rapport-conformite-rgpd-aipd-genomepath.md)
- [Audit RGPD Rocicorp Pharma](rapport-audit-rgpd-rocicorp-pharma.md)

## Catégories

### 🌐 Sécurité Web
Techniques d'audit et d'exploitation sur les applications web.

- [Audit IDOR](web/idor.md)
- [Lab Decima / Samaritan OS (Contrôle d'accès)](web/decima-samaritan-controle-acces.md)
- [Pentest API IBC-News (Grey Box)](web/ibc-news-api.md)
- [Contournement CSRF](web/csrf.md)
- [Bypass Upload de Fichiers](web/upload-bypass.md)
- [Fuzzing avec ffuf](web/ffuf.md)

### 🛡️ SIEM & SOC
Architecture, déploiement et gestion d'infrastructure de supervision.

- [Agent Wazuh – Déploiement multi-OS](siem-soc/agent-wazuh.md)
- [Analyse de Risques Cyber](siem-soc/analyse-risques.md)

### 🎯 Pentest
Méthodologies d'audit offensif de bout en bout, sur des cas réels.

- [Audit Combiné Red/Blue Team (Cas Cylentra Robotics)](pentest/cylentra-red-blue-team.md)

### 🗝️ Active Directory
Attaques et durcissement de l'annuaire Windows.

- [Kerberoasting](active-directory/kerberoasting.md)
- [Empoisonnement LLMNR & Relais NTLM](active-directory/llmnr-ntlm-relay.md)
- [Pass-the-Hash (PtH)](active-directory/pass-the-hash.md)

### 📚 Bibliothèque
Fiches opérationnelles et référentiels techniques transverses.

- [Briefing - Infrastructure de Détection et Gestion des Logs avec Wazuh](bibliotheque/wazuh-logs-infrastructure.md)
- [Fiche de Méthode - Le Flux de Travail Metasploit](bibliotheque/metasploit-workflow.md)
- [Glossaire Conceptuel - Les 6 Fonctions Vitales du NIST CSF 2.0](bibliotheque/nist-csf-2-fonctions-vitales.md)
- [Guide Fondamental - La Sainte Trinité du Réseau Windows (AD, DNS, DHCP)](bibliotheque/reseau-windows-ad-dns-dhcp.md)
- [Introduction aux Vulnérabilités de Configuration Active Directory](bibliotheque/ad-vulnerabilites-configuration.md)
- [Manuel de Procédures Forensiques - Acquisition et Analyse Multi-Vecteurs](bibliotheque/forensique-acquisition-analyse-multivecteurs.md)
- [Manuel de Procédures Opérationnelles - Réponse aux Compromissions d'Identifiants (Phishing)](bibliotheque/reponse-compromission-identifiants-phishing.md)
- [Protocole d'Investigation Numérique - Analyse de la Mémoire et Audit des Pilotes](bibliotheque/investigation-memoire-audit-pilotes.md)
- [Protocole d'Investigation Numérique - Détection d'Intrusion et Surveillance des Processus Critiques](bibliotheque/investigation-intrusion-processus-critiques.md)
- [Protocole d'Audit - Identification et Prévention de l'Escalade de Privilèges via les Services Windows](bibliotheque/escalade-privileges-services-windows.md)
- [Référentiel Technique - Architecture et Automatisation des Group Policy Objects (GPO)](bibliotheque/gpo-architecture-automatisation.md)
- [Investigation Réseau via Wireshark (Cas BookWorld)](bibliotheque/investigation-reseau-wireshark-bookworld.md)
- [Forensics Mémoire avec Volatility (Cas Weyland-Yutani)](bibliotheque/forensics-memoire-volatility-weyland-yutani.md)
- [DFIR Compromission WordPress (Cas Blue Sun Corporation)](bibliotheque/dfir-compromission-wordpress-blue-sun.md)
- [Analyse Statique avec Ghidra](bibliotheque/ghidra-analyse-statique.md)
