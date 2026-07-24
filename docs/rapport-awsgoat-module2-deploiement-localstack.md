# Rapport méthodologique — AWSGoat Module 2 : Tentative de déploiement local (LocalStack)

**Analyste :** Jean
**Date :** 24/07/2026
**Contexte :** Jedha Cyber Lead — Module 2 AWSGoat (SQLi → File Upload → ECS Breakout → IAM Privilege Escalation)
**Environnement :** Local (Windows, Docker Desktop), LocalStack Community + Personal Auth Token

---

## Objectif de la session

Cloner le repo AWSGoat, déployer l'infrastructure du Module 2 via Terraform/LocalStack en local (Docker Desktop), afin de préparer la réalisation du scénario d'attaque décrit dans `attack-manuals/module-2/`.

---

## Étapes réalisées

### 1. Clonage du repo

```
git clone https://github.com/old-dov/AWSGoat.git
```

### 2. Reconnaissance de la structure du repo

```
cd AWSGoat
Get-ChildItem
Get-ChildItem modules
Get-ChildItem modules\module-2
Get-ChildItem attack-manuals
Get-ChildItem attack-manuals\module-2
```

Structure confirmée : `modules/module-2/main.tf` (Terraform), `attack-manuals/module-2/` contient 4 manuels (SQL Injection, File Upload and Task Metadata, ECS Breakout and Instance Metadata, IAM Privilege Escalation).

### 3. Lecture du manuel d'attaque — étape 1 (SQL Injection)

```
Get-Content "attack-manuals\module-2\01-SQL Injection.md"
```

Résumé clé : injection sur le champ Email (`'or '1'='1'#`), pivot entre profils (Normal User / Manager / Admin) via `LIMIT` / `ORDER BY ... DESC` sur le champ `id`.

### 4. Préparation de l'environnement LocalStack (local, Docker Desktop)

```
docker info
docker run -d --name localstack -p 4566:4566 -p 4510-4559:4510-4559 localstack/localstack
docker ps -a --filter "name=localstack"
```

Conflit détecté avec un conteneur `localstack` préexistant (issu d'une session précédente, ports non publiés).

```
docker rm -f localstack
docker run -d --name localstack -p 4566:4566 -p 4510-4559:4510-4559 localstack/localstack
docker logs localstack
```

Échec : `Exited (55)` — licence LocalStack non activée (`LOCALSTACK_AUTH_TOKEN` manquant).

Réutilisation du Personal Auth Token existant (créé lors d'une session précédente, compte gratuit `app.localstack.cloud`) :

```
docker rm -f localstack
docker run -d --name localstack -p 4566:4566 -p 4510-4559:4510-4559 -e LOCALSTACK_AUTH_TOKEN=<token> localstack/localstack
docker ps --filter "name=localstack"
```

Conteneur opérationnel, statut `healthy`, licence activée.

### 5. Installation et configuration des outils clients

```
terraform --version
tflocal --version
aws --version
winget install Amazon.AWSCLI
aws configure --profile localstack
aws --endpoint-url=http://localhost:4566 sts get-caller-identity --profile localstack
```

Profil AWS CLI factice (`test`/`test`, région `us-east-1`) configuré et validé contre LocalStack.

```
pip show terraform-local
pip install terraform-local
```

Installation de `terraform-local` (tflocal 0.26.0) manquant sur le poste Windows.

### 6. Vérification des dépendances pour les provisioners Terraform

Le fichier `main.tf` contient deux ressources `null_resource` avec `provisioner "local-exec"` appelant `/bin/bash` et `sed`. Vérification de la disponibilité d'un interpréteur Bash compatible :

```
Get-Command bash
wsl -l -v
wsl -d Ubuntu -- which sed
```

Confirmation : WSL2 (distribution Ubuntu) fournit `/bin/bash` et `sed`, compatible avec les provisioners du fichier Terraform.

### 7. Lecture complète de `main.tf`

```
Get-Content main.tf
```

Ressources identifiées : VPC, subnets, Internet Gateway, route tables, security groups, RDS (MySQL), IAM (rôles/policies), ECS (cluster, task definition, service, ASG, launch template), ALB (load balancer, target group, listener), Secrets Manager, S3 (bucket état Terraform).

### 8. Initialisation et plan Terraform — premier essai

```
tflocal init
tflocal plan
```

Échec : erreurs `Unsupported argument` sur de nombreux endpoints (`bedrock`, `ce`, `codeconnections`, `deploy`, `dsql`, `keyspaces`, `logs`, `opensearch`, `pipes`, `s3tables`, `scheduler`, `verifiedpermissions`...).

### 9. Diagnostic de la cause

```
Get-Content .terraform.lock.hcl | Select-String "version"
```

Provider AWS verrouillé en version `3.76.1` (contrainte `~> 3.27` dans `main.tf`), trop ancien pour connaître les endpoints générés automatiquement par `tflocal` (liste exhaustive des services LocalStack actuels, sans filtrage par version de provider sauf 2 exceptions codées en dur).

Lecture du script source `tflocal` (`pip show -f terraform-local`, lecture du fichier `tflocal` dans `Scripts/`) pour confirmer le mécanisme de génération du fichier `localstack_providers_override.tf`.

### 10. Correction — assouplissement de la contrainte de version du provider

Modification manuelle de `main.tf` :

```hcl
# avant
version = "~> 3.27"
# après
version = ">= 5.0"
```

```
tflocal init -upgrade
tflocal plan
```

Plan généré avec succès, aucune erreur d'endpoint.

### 11. Apply — échec par limitation de licence

```
tflocal apply
```

Échec partiel : 4 services renvoient une erreur HTTP 501 (« not included within your LocalStack license ») — **RDS**, **Auto Scaling**, **ECS**, **ELBv2**. Ces services sont Pro-only sur le plan gratuit LocalStack. 29 ressources ont néanmoins été créées avant l'échec (VPC, IAM, security groups, Secrets Manager, S3, subnets, IGW, route tables).

### 12. Nettoyage

```
tflocal destroy
```

29 ressources détruites avec succès. Environnement LocalStack propre.

---

## Constat final

Le Module 2 d'AWSGoat repose sur 4 services AWS non disponibles sur le plan LocalStack Community/gratuit (RDS, ECS, Auto Scaling, ELBv2). Le déploiement complet — et donc la réalisation du scénario d'attaque (l'application cible tourne sur ECS derrière un ALB) — n'est pas réalisable en l'état sur ce plan.

Piste écartée : un vrai compte AWS n'est pas envisageable actuellement (moyen de paiement refusé sur AWS).
Piste écartée : OpenStack n'est pas une alternative viable — c'est une plateforme IaaS auto-hébergée sans compatibilité API AWS, incompatible avec le code Terraform existant (`aws_ecs_cluster`, `aws_db_instance`, etc.) et avec la logique du scénario d'attaque (task metadata ECS, IAM AWS).

## Pistes à explorer en prochaine session

- Vérifier les conditions d'un essai LocalStack Pro (durée, moyen de paiement requis ou non).
- Vérifier si une version antérieure du repo AWSGoat propose une infra plus légère (sans ECS/RDS/ALB) pour ce module.
- Évaluer si le scénario peut être rejoué manuellement sans Terraform, en recréant une application PHP+MySQL minimaliste en conteneurs Docker simples (sans passer par les services AWS émulés Pro-only), en s'appuyant uniquement sur le code source applicatif du repo (dossier `resources`/`src` de `module-2`) plutôt que sur l'infrastructure AWS complète.

---

## État de l'environnement en fin de session

- Conteneur `localstack` : actif, healthy, aucune ressource déployée (destroy complet).
- Repo AWSGoat cloné localement : `C:\Users\muzee\AWSGoat`.
- `main.tf` du Module 2 modifié (contrainte provider AWS `>= 5.0` au lieu de `~> 3.27`) — modification locale, à conserver pour la prochaine tentative.
- Outils installés et fonctionnels : Terraform 1.15.8, tflocal 0.26.0, AWS CLI 2.36.7, profil `localstack` configuré, WSL2 Ubuntu (bash/sed) disponible.
