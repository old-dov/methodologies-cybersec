# Rapport d'Audit RGPD — Rocicorp Pharma (Project Panacea)

**Analyste :** Jean
**Date :** 28 juillet 2026
**Mandat :** Audit technique White Box — Autorité de contrôle (simulation Jedha Cyber Lead)
**Cible :** Environnement de réplique production Rocicorp Pharma (jedha-cli, lab `rocicorp-pharma`)

---

## 1. Synthèse exécutive

L'audit a couvert quatre microservices de la plateforme "Project Panacea" et a permis de confirmer quatre violations actives du RGPD, chacune corrigée in situ par patch du code source Python :

| # | Target | Violation | Article RGPD | Statut |
|---|--------|-----------|---------------|--------|
| 1 | privacy-api | IDOR sur export DSAR | Art. 5(1)(f), Art. 32 | Corrigé |
| 2 | ai-engine | Décision automatisée sans recours | Art. 22 | Corrigé |
| 3 | db-postgres / kms-vault | Effacement impossible sur backup WORM | Art. 17 | Corrigé (crypto-shredding) |
| 4 | siem-logger | Absence de détection de brèche | Art. 33 | Corrigé |

---

## 2. Cadre réglementaire mobilisé

- **Amende administrative maximale (négligence grave)** : 4 % du chiffre d'affaires mondial annuel (Art. 83§5), ou 20 M€ si supérieur.
- **Notification de violation à l'autorité de contrôle sous 72h** : Article 33.
- **Transfert de données hors UE sans décision d'adéquation (post-Schrems II)** : Clauses Contractuelles Types (CCT / SCC).
- **Pseudonymisation et chiffrement des systèmes** : Article 32.
- **Droit d'opt-out sur la vente/partage de données (Californie)** : CPRA.

---

## 3. Target 1 — Leaky Portal (privacy-api)

### 3.1 Compliance Status
L'endpoint `/api/dsar/export` acceptait un paramètre `patient_id` fourni librement par le client, sans vérification de correspondance avec l'utilisateur authentifié. Faille de type IDOR (Insecure Direct Object Reference / CWE-639).

### 3.2 Legal Context
- **Article 5(1)(f)** — intégrité et confidentialité des données.
- **Article 32** — sécurité du traitement, contrôle d'accès défaillant.
- Données exposées : identité, email, IBAN, notes médicales chiffrées.

### 3.3 Technical Evidence

**Baseline légitime (patient_id=1001, utilisateur Alice Smith) :**
```
GET /api/dsar/export?patient_id=1001
```

**Exploitation (accès non autorisé aux données de Bob Jones) :**
```
GET /api/dsar/export?patient_id=1002
```

Résultat : retour intégral du dossier de Bob Jones (patient_id=1002) en étant authentifié en tant qu'Alice — confirmation de l'IDOR.

### 3.4 Remediation Details

Fichier corrigé : `/app/main.py`

```python
CURRENT_AUTHENTICATED_PATIENT_ID = "1001"

@app.get("/api/dsar/export")
def export_patient_data(patient_id: str):
    if patient_id != CURRENT_AUTHENTICATED_PATIENT_ID:
        raise HTTPException(status_code=403, detail="Forbidden: you can only access your own data.")
    conn = get_db_connection()
    if not conn:
        raise HTTPException(status_code=500, detail="Internal Server Error: DB unreachable")
    cursor = conn.cursor(cursor_factory=RealDictCursor)
    cursor.execute("SELECT patient_id, full_name, email, iban, encrypted_medical_notes FROM patients_pii WHERE patient_id = %s", (patient_id,))
    patient_data = cursor.fetchone()
    cursor.close()
    conn.close()
    if patient_data:
        return {
            "status": "success",
            "legal_basis": "GDPR Article 20 - Right to Data Portability",
            "data": patient_data
        }
    raise HTTPException(status_code=404, detail="Patient record not found.")
```

**Commandes d'application (SSH sur privacy-api, port 22) :**
```
cp /app/main.py /app/main.py.bak
grep -n '^}$' /app/main.py
sed -i '21a\
    if patient_id != CURRENT_AUTHENTICATED_PATIENT_ID:\
        raise HTTPException(status_code=403, detail="Forbidden: you can only access your own data.")' /app/main.py
kill 1
```

**Vérification post-patch :**
```
GET /api/dsar/export?patient_id=1001
GET /api/dsar/export?patient_id=1002
```
→ `patient_id=1001` : 200 (accès légitime maintenu)
→ `patient_id=1002` : 403 Forbidden (faille corrigée)

---

## 4. Target 2 — Algorithmic Cruelty (ai-engine)

### 4.1 Compliance Status
L'endpoint `/api/trial/evaluate` rejetait automatiquement les candidats selon des critères hardcodés (`age > 65` ou `"Diabetes Type 2"` dans l'historique médical), sans intervention humaine ni possibilité de recours.

### 4.2 Legal Context
- **Article 22** — droit de ne pas faire l'objet d'une décision exclusivement automatisée produisant des effets significatifs, sans droit d'obtenir une intervention humaine, d'exprimer son point de vue et de contester la décision.
- Discrimination potentielle de groupes vulnérables (âge, condition médicale).

### 4.3 Technical Evidence

**Test déclenchant le rejet automatique :**
```
POST /api/trial/evaluate
{"age": 78, "blood_type": "O+", "medical_history": ["hypertension"]}
```
Résultat avant patch : `status: REJECTED`, `human_review_possible: false`, message "No appeal is possible."

### 4.4 Remediation Details

Fichier corrigé : `/app/main.py`

```python
@app.post("/api/trial/evaluate")
def evaluate_patient(profile: PatientProfile):
    if profile.age > 65 or "Diabetes Type 2" in profile.medical_history:
        print(f"[AI FLAG] Patient profile flagged for human review. age={profile.age}, medical_history={profile.medical_history}, criteria_triggered=age>65 or Diabetes Type 2")
        return {
            "status": "PENDING_REVIEW",
            "reason": "Automated AI profiling flagged this case for mandatory human review",
            "human_review_possible": True,
            "message": "Your profile requires manual review by a doctor before a final decision is made."
        }
    return {
        "status": "APPROVED",
        "human_review_possible": False,
        "message": "You are eligible for the clinical trial."
    }
```

**Commandes d'application (SSH sur ai-engine, port 22) :**
```
cp /app/main.py /app/main.py.bak
grep -n "Generates a JSON export" /app/main.py
sed -i '22,29d' /app/main.py
sed -i '21a\
    if profile.age > 65 or "Diabetes Type 2" in profile.medical_history:\
        print(f"[AI FLAG] Patient profile flagged for human review. age={profile.age}, medical_history={profile.medical_history}, criteria_triggered=age>65 or Diabetes Type 2")\
        return {\
            "status": "PENDING_REVIEW",\
            "reason": "Automated AI profiling flagged this case for mandatory human review",\
            "human_review_possible": True,\
            "message": "Your profile requires manual review by a doctor before a final decision is made."\
        }' /app/main.py
kill 1
```

**Vérification post-patch :**
```
POST /api/trial/evaluate
{"age": 78, "blood_type": "O+", "medical_history": ["hypertension"]}
```
→ `status: PENDING_REVIEW`, `human_review_possible: true`, escalade vers révision humaine confirmée.

---

## 5. Target 3 — Illusion of Erasure (db-postgres / kms-vault)

### 5.1 Compliance Status
Les notes médicales des patients sont stockées chiffrées en base PostgreSQL et répliquées sur des backups WORM (immuables). Chaque patient dispose d'une clé de chiffrement individuelle gérée par le KMS. Aucun mécanisme de crypto-shredding n'était exercé au moment de l'audit.

### 5.2 Legal Context
- **Article 17** — droit à l'effacement.
- Le crypto-shredding constitue l'équivalent légal reconnu de l'effacement lorsque la suppression physique des données est techniquement impossible (backups WORM).

### 5.3 Technical Evidence

**Extraction des notes chiffrées (base `privacy_lab`) :**
```
python3 -c "
import psycopg2
conn = psycopg2.connect(host='10.0.15.13', port='5432', dbname='privacy_lab', user='admin', password='labpassword')
cur = conn.cursor()
cur.execute('SELECT patient_id, full_name, encrypted_medical_notes FROM patients_pii')
for row in cur.fetchall():
    print(row)
cur.close()
conn.close()
"
```

**Vérification de la clé active avant destruction (patient Bob Jones, patient_id=1002) :**
```
GET /api/kms/keys/1002
```
Résultat : clé active `key_kL9m2Q_rocicorp_secure`.

**Destruction de la clé (crypto-shredding) :**
```
DELETE /api/kms/keys/1002
```

### 5.4 Remediation Details

Aucune modification de code requise — le mécanisme de crypto-shredding est nativement implémenté côté KMS. La remédiation consiste en l'exécution effective de la procédure sur demande d'effacement (Article 17).

**Vérification post-destruction :**
```
GET /api/kms/keys/1002
```
Résultat : 404, "The data is crypto-shredded and permanently unreadable."

Conséquence : les `encrypted_medical_notes` du patient dans PostgreSQL et dans les backups WORM restent physiquement présentes mais sont désormais définitivement irrécupérables — effacement effectif au sens légal sans altération des backups immuables.

---

## 6. Target 4 — Silent Watcher (siem-logger)

### 6.1 Compliance Status
Le SIEM collectait passivement les logs sans aucune logique de détection d'anomalie. L'attaque IDOR réalisée au Target 1 (accès en rafale à plusieurs dossiers patients depuis une même IP) ne déclenchait aucune alerte.

### 6.2 Legal Context
- **Article 33** — obligation de notification à l'autorité de contrôle sous 72h en cas de violation de données. Cette obligation suppose une capacité de détection effective, absente ici.

### 6.3 Technical Evidence

**Simulation du pattern d'exfiltration (4 exports DSAR, même IP, fenêtre de 15s) :**
```
POST /api/logs/ingest
{"timestamp": 1753700000, "ip_address": "203.0.113.42", "service": "privacy-api", "action": "dsar_export", "status": "success", "details": "patient_id=1001"}
POST /api/logs/ingest
{"timestamp": 1753700005, "ip_address": "203.0.113.42", "service": "privacy-api", "action": "dsar_export", "status": "success", "details": "patient_id=1002"}
POST /api/logs/ingest
{"timestamp": 1753700010, "ip_address": "203.0.113.42", "service": "privacy-api", "action": "dsar_export", "status": "success", "details": "patient_id=1003"}
POST /api/logs/ingest
{"timestamp": 1753700015, "ip_address": "203.0.113.42", "service": "privacy-api", "action": "dsar_export", "status": "success", "details": "patient_id=1004"}
```

**Analyse avant patch :**
```
GET /api/logs/analyze
```
Résultat : `alert_level: GREEN`, "No breach detected" — absence totale de détection malgré le pattern d'exfiltration.

### 6.4 Remediation Details

Fichier corrigé : `/app/main.py`

```python
@app.get("/api/logs/analyze")
def analyze_logs_for_breach():
    THRESHOLD = 3
    WINDOW_SECONDS = 60

    ip_timestamps = {}
    for log in SECURITY_LOGS:
        if log["action"].lower() == "dsar_export":
            ip_timestamps.setdefault(log["ip_address"], []).append(log["timestamp"])

    for ip, timestamps in ip_timestamps.items():
        timestamps.sort()
        for i in range(len(timestamps)):
            window = [t for t in timestamps if t >= timestamps[i] and t <= timestamps[i] + WINDOW_SECONDS]
            if len(window) > THRESHOLD:
                return {
                    "alert_level": "CRITICAL",
                    "message": "Article 33 Triggered: 72h CNIL Notification Required!",
                    "source_ip": ip,
                    "event_count": len(window),
                    "detail": f"Data exfiltration pattern detected: {len(window)} DSAR exports from {ip} within {WINDOW_SECONDS}s"
                }

    return {
        "alert_level": "GREEN",
        "message": "All systems operational. No breach detected."
    }
```

**Commandes d'application (SSH sur siem-logger, port 22) :**
```
cp /app/main.py /app/main.py.bak
find / -name "main.py" 2>/dev/null
cat /app/main.py
cat > /app/main.py << 'EOF'
[contenu complet du fichier patché]
EOF
kill 1
```

**Vérification post-patch (réingestion des 4 logs puis analyse) :**
```
POST /api/logs/ingest [x4, mêmes payloads que ci-dessus]
GET /api/logs/analyze
```
Résultat : `alert_level: CRITICAL`, `source_ip: 203.0.113.42`, `event_count: 4`, déclenchement explicite de l'obligation de notification 72h.

---

## 7. Conclusion

Les quatre violations identifiées couvrent l'ensemble du cycle de vie de la donnée patient chez Rocicorp Pharma : accès (Target 1), traitement décisionnel (Target 2), effacement (Target 3), et détection/notification (Target 4). Chaque correctif a été appliqué directement sur le code source et validé par un test de non-régression (accès légitime toujours fonctionnel) et un test de confirmation (faille effectivement bloquée ou détectée).

**Recommandations complémentaires non couvertes par cet audit :**
- Remplacement de l'identité hardcodée (`CURRENT_AUTHENTICATED_PATIENT_ID`) par une authentification JWT/session réelle en production.
- Persistance des logs SIEM (actuellement en mémoire, perdus au redémarrage du service).
- Automatisation de la notification CNIL sous 72h suite à une alerte CRITICAL (actuellement seule la détection est implémentée).

---

*Rapport rédigé par Jean — bibliothèque méthodologique HIVE.*
