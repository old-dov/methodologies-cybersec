# Protocole d'Audit Active Directory : Pass-the-Hash (PtH)

## 1. Synthèse exécutive

Une fois un accès initial obtenu sur une machine du domaine (ex. `SRV1`), l'absence de restriction sur les mouvements latéraux et l'usage de protocoles d'administration à distance non durcis permettent une attaque de type **Pass-the-Hash (PtH)**. Cette technique étend la compromission à d'autres serveurs du réseau (ex. `SRV2`, voire le Contrôleur de Domaine `DC1`) **sans jamais avoir besoin de découvrir le mot de passe en clair** des comptes visés.

## 2. Mécanisme technique

Lorsqu'un utilisateur Windows s'authentifie, son mot de passe est converti en une empreinte cryptographique, le **hash NTLM**, conservée dans l'espace mémoire du processus `lsass.exe` (Local Security Authority Subsystem Service) pour permettre le Single Sign-On (SSO).

Le problème de fond : pour s'authentifier auprès d'un service distant via **SMB** ou **WinRM**, Windows n'exige pas le mot de passe en clair — la simple présentation d'un **hash NTLM valide suffit** à prouver l'identité.

## 3. Déroulement de l'exploitation

1. **Extraction** : avec les privilèges `NT AUTHORITY\SYSTEM` sur la machine compromise, un outil comme **Mimikatz** exécute `privilege::debug` puis `sekurlsa::logonpasswords` pour lire la mémoire de `lsass.exe` et en extraire le hash NTLM d'un compte à privilèges (ex. un administrateur du domaine).
2. **Authentification distante** : depuis la machine attaquante, le protocole **WinRM** (port 5985/5986) est ciblé via l'outil `evil-winrm`.
3. **Le pivot** : injection directe du hash via le paramètre `-H`. Le serveur distant valide la session sans aucune tentative de cassage de mot de passe.

```
# Extraction (sur la machine compromise, privilèges SYSTEM)
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords

# Pivot Pass-the-Hash vers un autre hôte du domaine
evil-winrm -i <IP_CIBLE> -u Administrator -H <HASH_NTLM>
```

**Résultat** : shell administratif complet sur la machine cible, sans qu'aucune alerte de cassage de mot de passe ne soit générée.

## 4. Détection

- **Event ID 4672** (attribution de privilèges élevés à l'ouverture de session) : à corréler avec une connexion WinRM/SMB depuis un hôte inhabituel.
- **Mouvements latéraux rapides** : authentifications successives d'un même compte à privilèges vers plusieurs hôtes en un temps très court.
- **Absence d'échec d'authentification préalable** : un PtH réussit généralement du premier coup, contrairement à une tentative de brute-force classique.

## 5. Plan de remédiation et durcissement

Le Pass-the-Hash n'est pas un bug logiciel mais un abus de conception des protocoles d'authentification hérités. La défense consiste à limiter la surface d'attaque en mémoire et à bloquer les mouvements latéraux.

### A. Microsoft LAPS (mesure prioritaire)

Les administrateurs réutilisent souvent le même mot de passe local sur toutes les machines : un hash volé sur `SRV1` ouvre alors instantanément `SRV2`. **LAPS (Local Administrator Password Solution)** génère un mot de passe local unique, complexe et aléatoire par machine, stocké de façon sécurisée dans l'AD. Le vol d'un hash local reste alors confiné à une seule machine.

### B. Restriction de la mémoire et groupes de sécurité

- **Groupe « Protected Users »** : y ajouter les comptes à hauts privilèges. Ce groupe interdit strictement la mise en cache des hashes NTLM et des clés Kerberos faibles en mémoire, rendant Mimikatz inefficace sur ces comptes.
- **Windows Defender Credential Guard** : à activer via GPO sur Windows 10/11 et Windows Server. Grâce à la virtualisation (VBS), `lsass.exe` est isolé dans un conteneur sécurisé qu'un administrateur local ne peut pas lire.

### C. Limitation du protocole NTLM

Configurer les GPO pour restreindre le trafic NTLM sortant et forcer l'usage exclusif de Kerberos, qui s'appuie sur des tickets à durée limitée et n'est pas vulnérable au Pass-the-Hash traditionnel.

## 6. Conclusion

Le Pass-the-Hash transforme un accès local isolé en compromission de domaine complète si aucune barrière n'est posée entre machines. LAPS et le groupe Protected Users constituent les deux mesures à plus fort impact immédiat pour confiner ce risque.
