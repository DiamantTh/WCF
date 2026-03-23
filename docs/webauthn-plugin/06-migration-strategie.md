# 06 – Migrations-Strategie: Plugin → Core

> Dieser Plan beschreibt, wie das WebAuthn-Plugin später in einen offiziellen WCF-Core-
> Einbau überführt werden kann, ohne dass bestehende Nutzerdaten verloren gehen.

---

## Voraussetzungen für eine Migration

Bevor eine Migration möglich ist, müssen folgende Fragen bekannt sein:

1. **Datenbankschema des Core-Einbaus** – Sind die Felder identisch? Gibt es Umbenennungen?
2. **objectType-Name im Core** – Heißt die Methode `com.woltlab.wcf.multifactor.webauthn`?
3. **Klassen-Namespace** – Wird `wcf\system\user\multifactor\WebAuthnMultifactorMethod` im Core verwendet?
4. **Format des Public Keys** – COSE-Rohdaten oder PEM?

---

## Migrations-Szenarien

### Szenario A: Schema ist identisch (ideal)

Das Plugin-Schema und das Core-Schema stimmen überein.
Vorgehen:

```sql
-- 1. Plugin-Daten in Core-Tabelle kopieren (falls umbenannt)
INSERT INTO wcf1_user_multifactor_webauthn_core
SELECT * FROM wcf1_user_multifactor_webauthn;

-- 2. objectTypeID aktualisieren (falls Core anderen objectType verwendet)
UPDATE wcf1_user_multifactor m
JOIN   wcf1_object_type ot ON ot.objectType = 'com.woltlab.wcf.multifactor.webauthn'
SET    m.objectTypeID = ot.objectTypeID
WHERE  m.objectTypeID = (
    SELECT objectTypeID FROM wcf1_object_type
    WHERE  objectType = 'com.example.wcf.multifactor.webauthn'
);

-- 3. Plugin-Tabelle löschen
DROP TABLE wcf1_user_multifactor_webauthn;

-- 4. Plugin deinstallieren (objectType-Eintrag + Klassen)
```

### Szenario B: Schema unterscheidet sich

Es sind Datentransformationen nötig (z.B. anderes Key-Format).

```sql
-- Beispiel: Konvertierung Public Key von PEM zu COSE
-- (In einem PHP-Migrations-Script, nicht in SQL)

$sql = "SELECT * FROM wcf1_user_multifactor_webauthn";
$statement = WCF::getDB()->prepare($sql);
$statement->execute();
while ($row = $statement->fetchArray()) {
    $cosePublicKey = convertPemToCose($row['publicKey']);

    $insertSql = "INSERT INTO wcf1_user_multifactor_webauthn_core
                              (setupID, credentialId, publicKey, signCount, deviceName, createTime)
                  VALUES      (?, ?, ?, ?, ?, ?)";
    // ...
}
```

### Szenario C: Core hat eigene objectTypeID

Wenn der Core einen anderen objectType-Namen verwendet, müssen die `setupID`-Verknüpfungen
migriert werden:

```sql
-- Schritt 1: Neue setupIDs für Core-objectType erstellen
INSERT INTO wcf1_user_multifactor (userID, objectTypeID)
SELECT m.userID, :coreObjectTypeID
FROM   wcf1_user_multifactor m
JOIN   wcf1_object_type ot ON ot.objectTypeID = m.objectTypeID
                          AND ot.objectType = 'com.example.wcf.multifactor.webauthn';

-- Schritt 2: Credential-Daten auf neue setupIDs umschreiben
UPDATE wcf1_user_multifactor_webauthn w
JOIN   wcf1_user_multifactor old ON old.setupID = w.setupID
JOIN   wcf1_user_multifactor new ON new.userID = old.userID
                                AND new.objectTypeID = :coreObjectTypeID
SET    w.setupID = new.setupID;

-- Schritt 3: Alte setupIDs entfernen
DELETE FROM wcf1_user_multifactor
WHERE  objectTypeID = :pluginObjectTypeID;
```

---

## Empfehlungen für Plugin-Entwicklung (zukunftssicher)

### 1. Standard-Feldnamen verwenden

Orientierung an den bestehenden MFA-Tabellen im Core:
- `setupID` (nicht `mfaSetupId` o.ä.)
- `createTime` (nicht `registrationDate`)
- `lastUsed` (nicht `lastUsedAt`)
- `deviceName` (nicht `credentialLabel`)

### 2. Binäre Credential-IDs speichern

```php
// ✓ Korrekt: Binär (VARBINARY)
$credentialIdBinary = base64_decode(str_replace(['-', '_'], ['+', '/'], $b64url));

// ✗ Falsch: String (führt zu Encoding-Problemen bei Migration)
$credentialIdString = $b64url;
```

### 3. Plugin-Paketname sorgfältig wählen

Wenn WoltLab einen Core-objectType `com.woltlab.wcf.multifactor.webauthn` einführt,
sollte das Plugin **nicht** denselben Namen verwenden:

```
Plugin:  com.example.wcf.multifactor.webauthn   ← eigener Namespace
Core:    com.woltlab.wcf.multifactor.webauthn    ← WoltLab-Namespace
```

### 4. Update-Skript vorbereiten

Im Plugin ein optionales Update-Skript vorhalten, das bei Erkennung des Core-objectTypes
automatisch migriert:

```php
// In einem plugin-update script:
$coreObjectType = ObjectTypeCache::getInstance()
    ->getObjectTypeByName('com.woltlab.wcf.multifactor', 'com.woltlab.wcf.multifactor.webauthn');

if ($coreObjectType !== null) {
    // Core hat WebAuthn eingebaut → migrieren
    MigrationHelper::migrateToCore($coreObjectType);
}
```

---

## Checkliste vor der Migration

- [ ] WCF-Core mit eingebautem WebAuthn installieren
- [ ] Core-Datenbankschema mit Plugin-Schema vergleichen
- [ ] Core-objectType-Name herausfinden
- [ ] Backup der Produktionsdatenbank erstellen
- [ ] Migrations-Skript auf Staging-System testen
- [ ] signCount-Werte sind korrekt migriert
- [ ] Alle User können sich nach Migration noch mit ihren Keys einloggen
- [ ] Plugin-Plugin-Tabelle gelöscht
- [ ] Plugin deinstalliert

---

## Warum diese Strategie?

Das Plugin wurde so entwickelt, dass die Daten **ohne Verlust** in einen Core-Einbau
überführt werden können. Der entscheidende Punkt:

> Die `wcf1_user_multifactor`-Elterntabelle mit `setupID` ist Core-Standard.
> Das Plugin fügt nur eine eigene Kindtabelle hinzu, die über `setupID` verknüpft ist.
> Diese Verknüpfungsstruktur ist identisch mit TOTP, Email und Backup-Codes.
> Eine Migration ist daher strukturell immer möglich.

Die Nutzerdaten (Public Keys, signCounts, Gerätenamen) gehen bei einer sorgfältigen
Migration **nicht verloren**. User müssen ihre Security Keys **nicht neu registrieren**.
