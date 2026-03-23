# 05 – Datenbankschema

> Vollständiges SQL-Schema für das WebAuthn-Plugin.
> Prefix `wcf1_` wird von WCF automatisch durch den konfigurierten Prefix ersetzt.

---

## Neue Tabelle: `wcf1_user_multifactor_webauthn`

```sql
-- ============================================================
-- WebAuthn/FIDO2 MFA Plugin
-- install.sql
-- ============================================================

CREATE TABLE IF NOT EXISTS wcf1_user_multifactor_webauthn (
    -- Verknüpfung zur zentralen MFA-Setup-Tabelle
    setupID         INT(10)          NOT NULL,

    -- Eindeutige ID des Credentials (Binär, aus rawId des Authenticators)
    -- Base64url-Länge kann variieren (z.B. 16–1024 Byte)
    credentialId    VARBINARY(1024)  NOT NULL,

    -- COSE-kodierter Public Key oder PEM-String
    -- Länge hängt vom Algorithmus ab (EC: ~77 Byte, RSA-2048: ~294 Byte)
    publicKey       BLOB             NOT NULL,

    -- Replay-Schutz: muss bei jeder Authentifizierung erhöht werden
    -- Wert 0 = Authenticator unterstützt keine Zählung (z.B. manche Passkeys)
    signCount       INT(10) UNSIGNED NOT NULL DEFAULT 0,

    -- Anzeigename für die Verwaltungsoberfläche
    deviceName      VARCHAR(255)     NOT NULL,

    -- Zeitstempel der Registrierung (Unix-Timestamp)
    createTime      INT(10)          NOT NULL,

    -- Zeitstempel der letzten Nutzung (Unix-Timestamp, NULL = noch nie genutzt)
    lastUsed        INT(10)          DEFAULT NULL,

    -- Optional: AAGUID des Authenticators (16 Byte UUID)
    -- Erlaubt spätere Erkennung des Authenticator-Typs
    aaguid          BINARY(16)       DEFAULT NULL,

    -- Primary Key auf Kombination Setup + CredentialId
    -- (ein User kann mehrere Credentials haben)
    UNIQUE KEY (setupID, credentialId)
);

-- Fremdschlüssel: Cascade-Löschen wenn User oder MFA-Setup entfernt wird
ALTER TABLE wcf1_user_multifactor_webauthn
    ADD FOREIGN KEY (setupID)
        REFERENCES wcf1_user_multifactor (setupID)
        ON DELETE CASCADE;
```

---

## Deinstallation: `uninstall.sql`

```sql
-- Wird bei Plugin-Deinstallation ausgeführt
-- Die wcf1_user_multifactor-Einträge werden durch WCF's objectType-System
-- automatisch entfernt, was via CASCADE auch diese Tabelle leert.
-- Daher hier nur DROP TABLE:

DROP TABLE IF EXISTS wcf1_user_multifactor_webauthn;
```

---

## Referenz: Bestehende MFA-Tabellen (Core)

### `wcf1_user_multifactor` (Elterntabelle – nicht verändern)

```sql
CREATE TABLE wcf1_user_multifactor (
    setupID      INT(10) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    userID       INT(10) NOT NULL,
    objectTypeID INT(10) NOT NULL,
    UNIQUE KEY (userID, objectTypeID)
);
```

### Verknüpfungsstruktur

```
wcf1_user
    │ userID
    ▼
wcf1_user_multifactor
    │ setupID ◄─── objectTypeID → wcf1_object_type (com.example.wcf.multifactor.webauthn)
    │
    ▼
wcf1_user_multifactor_webauthn
    │ setupID, credentialId, publicKey, signCount, ...
```

---

## Feld-Erläuterungen

### `credentialId` (VARBINARY)

Die Credential-ID ist eine opake Byte-Sequenz, die vom Authenticator generiert wird.
- **Typ:** Binär (nicht als Text speichern – Encoding-Probleme!)
- **Länge:** Laut WebAuthn-Spec max. 1023 Byte, typisch 16–64 Byte
- **Herkunft:** `credential.rawId` aus der Browser-API

```php
// Korrekte Speicherung:
$credentialIdBinary = base64_decode(str_replace(['-', '_'], ['+', '/'], $base64urlValue));
$statement->execute([$credentialIdBinary]);

// Korrekte Abfrage:
WHERE credentialId = ?  // Mit Binärwert
```

### `publicKey` (BLOB)

Der Public Key wird vom Authenticator in COSE-Format geliefert.
- **Option A:** COSE-Rohdaten speichern (kompakt, benötigt COSE-Decoder bei Verifikation)
- **Option B:** Als PEM-String konvertiert speichern (einfacher für `openssl_verify`)

Empfehlung: **PEM-Format** für einfachere Signaturprüfung mit PHPs OpenSSL-Extension.

### `signCount` (INT UNSIGNED)

Replay-Schutz-Zähler:
- Wird vom Authenticator bei jeder Nutzung erhöht
- Server MUSS sicherstellen: neuer Wert > alter Wert (außer beide = 0)
- Bei Wert 0: Authenticator unterstützt keine Zählung → Replay-Angriff nicht erkennbar
- **Kein** automatisches Inkrementieren durch Server wenn Wert 0 ist (laut WebAuthn-Spec)

### `aaguid` (BINARY(16))

Authenticator Attestation GUID – identifiziert den Authenticator-Typ:
- Optional speichern, nützlich für spätere Filterung/Anzeige
- Bekannte AAGUIDs: [https://www.iana.org/assignments/webauthn/webauthn.xhtml](https://www.iana.org/assignments/webauthn/webauthn.xhtml)
- Beispiel: YubiKey 5 = `2fc0579f-8113-47ea-b116-bb5a8db9202a`

---

## Index-Empfehlungen

```sql
-- Für schnelle Lookup by credentialId (Authentication):
-- Bereits durch UNIQUE KEY (setupID, credentialId) abgedeckt

-- Für Verwaltungsansicht (Liste aller Credentials eines Users):
-- Bereits durch den JOIN über setupID → userID abgedeckt

-- Optional: Index für credential-only Lookup (ohne setupID)
-- Kann nützlich sein wenn credentialId global eindeutig sein soll:
-- ALTER TABLE wcf1_user_multifactor_webauthn ADD UNIQUE KEY idx_credentialId (credentialId);
-- ACHTUNG: Spec garantiert keine globale Eindeutigkeit – deshalb nicht standardmäßig
```

---

## Migrations-Kompatibilität

Das Schema wurde so entworfen, dass es später mit einem offiziellen WCF-Core-Schema
kompatibel sein kann:

| Feld | Voraussichtlich im Core | Kompatibel? |
|------|------------------------|-------------|
| `setupID` | Ja (kommt aus Core-Tabelle) | ✓ |
| `credentialId` | Ja | ✓ (VARBINARY) |
| `publicKey` | Ja | ✓ (BLOB) |
| `signCount` | Ja | ✓ |
| `deviceName` | Wahrscheinlich | ✓ |
| `createTime` | Wahrscheinlich | ✓ |
| `lastUsed` | Wahrscheinlich | ✓ |
| `aaguid` | Optional | ✓ |

Siehe [06-migration-strategie.md](06-migration-strategie.md) für den Migrationsplan.
