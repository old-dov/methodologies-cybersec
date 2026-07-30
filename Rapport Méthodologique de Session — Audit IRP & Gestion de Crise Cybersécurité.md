# **Rapport Méthodologique de Session**

#  **—** 

# **Audit IRP & Gestion de Crise Cybersécurité**

**Date :** 30 Juillet 2026

**Document à l'attention de :** Direction de la Sécurité des Systèmes d'Information (DSSI)

**Objet :** Synthèse méthodologique et retours d'expérience sur les audits de plans de réponse aux incidents (Tyrell Corporation) et la résolution de crise de ransomware (IBC Bank).

## **1\. Cadre Général et Approche Méthodologique**

Cette session a suivi un modèle de réflexion logique, séquentiel et étape par étape. Chaque phase d'analyse et de décision a été soumise à validation formelle avant d'engager la suivante.

L'approche retenue repose sur les principes suivants :

* **Interdiction des actions destructives de preuves :** Primauté de la préservation forensique (maintien sous tension, captures RAM, isolation logique vs extinction brute).  
* **Alignement réglementaire strict :** Intégration systématique des obligations légales (RGPD Articles 33 et 34, PCI DSS, réglementations bancaires).  
* **Arbitrage Sécurité vs Métier :** Neutralisation prioritaire des menaces actives avec prise en compte des contraintes opérationnelles (procédures dégradées).  
* **Communication hors-bande :** Étanchéité absolue des canaux de crise en cas de compromission des identifiants d'administration du domaine.

## 

## 

## **2\. Synthèse du Cas 1 — Audit du Plan IRP (Tyrell Corporation)**

L'évaluation de l'Incident Response Plan (v2.1) de Tyrell Corporation a permis d'identifier quatre faiblesses structurelles majeures et de formuler les recommandations associées.

### **Synthèse des constats**

* **Gouvernance et Rôles :** Absence du DPO et du service juridique au sein de l'IRT ; cumul de casquettes à risque (Technical Lead \= Head of Infrastructure) engendrant des conflits d'intérêts opérationnels.  
* **Détection :** Dépendance exclusive aux logs réseau et SIEM ; absence de solution EDR et de filtrage dynamique du vecteur e-mail.  
* **Confinement :** Autorisation d'extinction brute des serveurs (*shutdown*), détruisant irrémédiablement la mémoire vive (RAM) et les preuves forensiques.  
* **Notification :** Omission du délai légal de 72h du RGPD (AEPD/CNIL) et absence de procédure de notification des personnes concernées (données génomiques).

### **Commandes et actions de remédiation formulées**

Bash  
\# Exemple de configuration d'urgence pour la préservation forensique (Memory Dump avant isolation)  
\# Capture mémoire à chaud via WinPmem / LiME avant toute modification d'état  
winpmem.exe \-o mem\_dump.raw \--volume\_shadow\_copies

\# Isolation réseau logique au niveau de l'hôte via pare-feu local (remplacement du shutdown)  
netsh advfirewall set allprofiles state on  
netsh advfirewall firewall add rule name="IR\_Isolation" dir=out action=block

## 

## 

## 

## 

## 

## **3\. Synthèse du Cas 2 — Gestion d'Incident Majeur Ransomware (IBC Bank)**

L'incident survenu à l'InterGalactic Banking Clan a fait l'objet d'un traitement opérationnel complet, du triage initial jusqu'à la production du rapport post-incident.

### **Chronologie et Triage de Crise**

1. **Accès Initial :** Phishing ciblé (3 mars, 10h32) via macro `.docx` exécutant un script PowerShell sur la machine `WKS-0347` (détection configurée en *log-only*).  
2. **Élévation de Privilèges :** Exécution de Mimikatz sur le contrôleur de domaine `DC-01` (absence de protection LSASS/LSA Protection).  
3. **Mouvement Latéral & Exfiltration :** Mouvement RDP non restreint du VLAN Utilisateur vers le VLAN Production ; exfiltration de 4,2 Go de données clients (35 000 dossiers) vers un serveur C2.  
4. **Impact :** Chiffrement par ransomware des serveurs de transaction (`TXNSERV-01`, `TXNSERV-02`) et des sauvegardes locales (`BKPSRV-01`).

### **Matrice des Décisions de Confinement**

| Système | Action Retenue | Rationale Technico-Légal |
| ----- | ----- | ----- |
| `WKS-0347` | **Isolation logique** | Patient zéro. Maintien sous tension pour extraction de la RAM. |
| `TXNSERV-01` | **Isolation logique** | Interruption du flux d'exfiltration HTTPS et du chiffrement, conservation des clés en mémoire. |
| `TXNSERV-02` | **Isolation logique** | Blocage de la propagation du ransomware sur le second nœud. |
| `BKPSRV-01` | **Isolation logique** | Protection des sauvegardes encore saines / hors ligne. |
| `WEBFRONT-01` | **Mise en maintenance** | Dépendance fonctionnelle coupée ; affichage d'une page d'attente client. |
| `DC-01` | **Révocation / Reset** | Maintien du service d'annuaire ; réinitialisation double du compte `krbtgt` et des accès Admin Domaine. |

## 

## 

## **4\. Bilan des Axes d'Amélioration Stratégiques (PIR-2025-003)**

Les causes racines identifiées débouchent sur quatre chantiers prioritaires à intégrer à la feuille de route de sécurité :

1. **Eradication des connexions directes inter-VLAN :** Implémentation d'un modèle Zero Trust exigeant des bastions d'administration (PAW) pour tout accès RDP/SSH vers la production.  
2. **Durcissement des Endpoints et de l'Identité :** Déploiement d'agents EDR en mode blLocal/Actif, activation de *LSA Protection* (`RunAsPPL`) sur tous les DCs et bascule des règles de détection d'exécutions de macros du mode *log-only* au mode *quarantine*.  
3. **Refonte de l'Architecture de Sauvegarde :** Mise en œuvre d'un stockage immuable (WORM / S3 Object Lock) hors du domaine Active Directory principal pour respecter la règle 3-2-1-1-0.  
4. **Automation de la Réponse (SOAR) :** Création de playbooks d'isolation automatique dès la détection de renommages massifs de fichiers ou d'exfiltrations anormales pour réduire le délai d'escalade.

*Fin du rapport méthodologique.*

