# 03 – MFA-Subsystem – Technische Tiefenanalyse

> Quellbasis: WoltLab Suite Core 6.2.2
> Eingeführt: WCF 5.4

---

## Inhaltsverzeichnis

- [Architekturübersicht](#architekturübersicht)
- [Datenbankschema (Core)](#datenbankschema-core)
- [IMultifactorMethod Interface](#imultifactormethod-interface)
- [Setup-Klasse](#setup-klasse)
- [MultifactorManageForm](#multifactormanageform)
- [MultifactorAuthenticationForm](#multifactorauthenticationform)
- [SessionHandler – MFA-relevante Methoden](#sessionhandler--mfa-relevante-methoden)
- [CheckForMultifactorRequirement Middleware](#checkformultifactorrequirement-middleware)
- [Vollständiger Login-Flow](#vollständiger-login-flow)
- [Eingebaute MFA-Methoden als Referenz](#eingebaute-mfa-methoden-als-referenz)

---

## Architekturübersicht

```
┌─────────────────────────────────────────────────────────────────┐
│                         Datenbank                                │
│                                                                   │
│  wcf1_user_multifactor (pro User + Methode)                     │
│  ┌──────────┬────────┬─────────────┐                            │
│  │ setupID  │ userID │ objectTypeID│                            │
│  └──────────┴────────┴─────────────┘                            │
│       │                                                           │
│       ▼                                                           │
│  wcf1_user_multifactor_webauthn (plugin-eigene Tabelle)         │
│  ┌──────────┬─────────────┬────────────┬──────────┐             │
│  │ setupID  │ credentialId│ publicKey  │signCount │             │
│  └──────────┴─────────────┴────────────┴──────────┘             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         PHP-Schicht                              │
│                                                                   │
│  Setup::find($objectType, $user)                                │
│      → gibt Setup-Objekt zurück (mit setupID)                   │
│                                                                   │
│  Setup::allocateSetUpId($objectType, $user)                     │
│      → erstellt neuen Setup-Eintrag                             │
│                                                                   │
│  IMultifactorMethod (Plugin implementiert dieses Interface)     │
│  ├── getStatusText(Setup)                                       │
│  ├── createManagementForm(IFormDocument, ?Setup, $returnData)   │
│  ├── processManagementForm(IFormDocument, Setup)                │
│  ├── createAuthenticationForm(IFormDocument, Setup)             │
│  └── processAuthenticationForm(IFormDocument, Setup)            │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         Forms                                    │
│                                                                   │
│  MultifactorManageForm        (Einrichtung / Verwaltung)        │
│  MultifactorAuthenticationForm (Login-Schritt 2)                │
│  MultifactorDisableForm       (Deaktivierung)                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Datenbankschema (Core)

### `wcf1_user_multifactor` (Zentrale MFA-Tabelle)

```sql
CREATE TABLE wcf1_user_multifactor (
    setupID     INT(10) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    userID      INT(10) NOT NULL,
    objectTypeID INT(10) NOT NULL,
    UNIQUE KEY (userID, objectTypeID)
);

ALTER TABLE wcf1_user_multifactor
    ADD FOREIGN KEY (userID)
        REFERENCES wcf1_user (userID) ON DELETE CASCADE;
ALTER TABLE wcf1_user_multifactor
    ADD FOREIGN KEY (objectTypeID)
        REFERENCES wcf1_object_type (objectTypeID) ON DELETE CASCADE;
```

**Bedeutung:**
- Ein Eintrag pro (User, MFA-Methode)-Kombination
- `setupID` = primärer Schlüssel für alle Plugin-Tabellen
- `ON DELETE CASCADE` = wenn User gelöscht wird, werden alle MFA-Daten automatisch entfernt

### Nutzer-Flag

```sql
-- In wcf1_user:
multifactorActive INT(1)  -- 1 wenn mind. eine MFA-Methode aktiv
```

### Plugin-eigene Tabellen

Jede MFA-Methode verwaltet eigene Tabellen, die über `setupID` mit der Haupttabelle verknüpft sind:

```sql
-- TOTP-Geräte
CREATE TABLE wcf1_user_multifactor_totp (
    setupID    INT(10) NOT NULL,
    deviceID   VARCHAR(191) NOT NULL,
    deviceName VARCHAR(255) NOT NULL,
    secret     VARBINARY(255) NOT NULL,
    minCounter INT(10) NOT NULL,
    createTime INT(10) NOT NULL,
    useTime    INT(10) DEFAULT NULL,
    UNIQUE KEY (setupID, deviceID)
);

-- Email-Codes
CREATE TABLE wcf1_user_multifactor_email (
    setupID    INT(10) NOT NULL,
    code       VARCHAR(191) NOT NULL,
    createTime INT(10) NOT NULL,
    UNIQUE KEY (setupID, code)
);

-- Backup-Codes
CREATE TABLE wcf1_user_multifactor_backup (
    setupID    INT(10) NOT NULL,
    identifier VARCHAR(191) NOT NULL,
    code       VARCHAR(255) NOT NULL,  -- bcrypt-Hash
    createTime INT(10) NOT NULL,
    useTime    INT(10) DEFAULT NULL,
    UNIQUE KEY (setupID, identifier)
);
```

---

## IMultifactorMethod Interface

**Dateipfad:** `wcfsetup/install/files/lib/system/user/multifactor/IMultifactorMethod.class.php`

```php
interface IMultifactorMethod
{
    /**
     * Gibt einen lesbaren Status-Text zurück, z.B. "3 Geräte registriert".
     * Wird in der Sicherheitsübersicht des Users angezeigt.
     */
    public function getStatusText(Setup $setup): string;

    /**
     * Füllt das Verwaltungs-Formular (Registrierung neuer Credentials).
     *
     * @param ?mixed $returnData  Wird nach processManagementForm() zurückgegeben
     *                             (z.B. Klartext-Codes, die nur einmal angezeigt werden)
     */
    public function createManagementForm(IFormDocument $form, ?Setup $setup, $returnData = null): void;

    /**
     * Verarbeitet das Verwaltungs-Formular.
     * Wird INNERHALB einer DB-Transaktion ausgeführt.
     *
     * MUSS sicherstellen, dass nach Abschluss ein gültiger DB-Zustand vorliegt
     * (d.h. mind. eine nutzbare Authentifizierungsmöglichkeit existiert).
     *
     * @return mixed  Opake Daten, die an createManagementForm() als $returnData übergeben werden
     */
    public function processManagementForm(IFormDocument $form, Setup $setup): mixed;

    /**
     * Füllt das Authentifizierungs-Formular (Login-Schritt 2).
     * Hier wird z.B. die WebAuthn-Challenge generiert und in der Session gespeichert.
     */
    public function createAuthenticationForm(IFormDocument $form, Setup $setup): void;

    /**
     * Verarbeitet das Authentifizierungs-Formular.
     * Wird INNERHALB einer DB-Transaktion ausgeführt.
     *
     * MUSS replay-sicher sein (z.B. signCount-Prüfung, Code-Invalidierung via DELETE/UPDATE).
     * MUSS eine Exception werfen wenn die Validierung fehlschlägt.
     *
     * @throws \RuntimeException Bei Validierungsfehler
     */
    public function processAuthenticationForm(IFormDocument $form, Setup $setup): void;
}
```

### Wichtige Implementierungsregeln

1. **Transaktionssicherheit:** Beide `process*`-Methoden laufen in DB-Transaktionen. Keine eigenen
   `beginTransaction()`-Aufrufe.

2. **Replay-Schutz:** `processAuthenticationForm()` MUSS den Credentials-Eintrag so modifizieren,
   dass dieselbe Response nicht zweimal akzeptiert wird (z.B. signCount > alter signCount).

3. **TOCTOU-Prüfung:** Formular-Validierung (in `create*`) und DB-Update (in `process*`) können
   voneinander abweichen. `process*` MUSS die Credentials nochmals aus der DB laden und mit
   `FOR UPDATE` sperren.

4. **Mindest-Credential:** `processManagementForm()` darf den User nicht in einen Zustand bringen,
   in dem er sich nicht mehr authentifizieren kann (= mind. 1 Credential muss übrig bleiben).

---

## Setup-Klasse

**Dateipfad:** `wcfsetup/install/files/lib/system/user/multifactor/Setup.class.php`

```php
final class Setup implements IIDObject
{
    // Gibt setupID zurück
    public function getId(): int;

    // Gibt den ObjectType zurück (= die MFA-Methoden-Konfiguration)
    public function getObjectType(): ObjectType;

    // Gibt den User zurück
    public function getUser(): User;

    // Sperrt den DB-Eintrag (FOR UPDATE), gibt $this zurück
    public function lock(): self;

    // Löscht den Setup-Eintrag (und alle verknüpften Plugin-Daten via CASCADE)
    public function delete(): void;

    // Findet einen bestehenden Setup für (objectType, user) oder null
    public static function find(ObjectType $objectType, User $user): ?self;

    // Gibt alle Setups eines Users zurück [setupID => Setup]
    public static function getAllForUser(User $user): array;

    // Erstellt einen neuen Setup-Eintrag und gibt ihn zurück
    public static function allocateSetUpId(ObjectType $objectType, User $user): self;
}
```

### Wichtige Details

- `Setup::find()` wird in `MultifactorManageForm::readParameters()` aufgerufen, um zu prüfen,
  ob der User diese Methode bereits eingerichtet hat (`$setup = null` → Ersteinrichtung)
- `Setup::lock()` gibt `$this` zurück und führt ein `SELECT ... FOR UPDATE` durch
- Das `MultifactorManageForm` ruft intern `Setup::allocateSetUpId()` auf, falls `$setup === null`

---

## MultifactorManageForm

**Dateipfad:** `wcfsetup/install/files/lib/form/MultifactorManageForm.class.php`

### Was der Core übernimmt (automatisch)

1. **Re-Authentifizierung erzwingen** (via `TReauthenticationCheck`)
2. **Backup-Codes automatisch generieren** bei erster MFA-Einrichtung
3. **`multifactorActive = 1`** beim User setzen
4. **Bestätigungs-E-Mail** senden
5. **DB-Transaktion** um `processManagementForm()`
6. **`Setup::allocateSetUpId()`** aufrufen wenn Setup noch nicht existiert

### Was das Plugin implementiert

- `createManagementForm()` → UI für Credential-Registrierung
- `processManagementForm()` → Credential-Validierung und -Speicherung

### Form-Flow

```
GET  /multifactor/manage?id={objectTypeID}
  → createManagementForm() wird aufgerufen mit $setup = null (Ersteinrichtung)
     oder $setup = Setup-Objekt (Verwaltung)

POST /multifactor/manage?id={objectTypeID}
  → Formular-Validierung
  → processManagementForm() in Transaktion
  → Bei Ersteinrichtung: Backup-Codes generieren, E-Mail senden
  → Redirect back (mit Success-Message via $returnData)
```

---

## MultifactorAuthenticationForm

**Dateipfad:** `wcfsetup/install/files/lib/form/MultifactorAuthenticationForm.class.php`

### Was der Core übernimmt

1. Lädt alle Setup-Instanzen des Users, sortiert nach `priority DESC`
2. Zeigt Methoden-Auswahl bei mehreren aktiven Methoden
3. `Setup::lock()` vor `processAuthenticationForm()`
4. `WCF::getSession()->applyPendingUserChange()` nach Erfolg
5. `UserLoggedIn`-Event feuern

### Methoden-Auswahl

```php
// Aus MultifactorAuthenticationForm::readParameters():
\uasort($this->setups, static function (Setup $a, Setup $b) {
    return $b->getObjectType()->priority <=> $a->getObjectType()->priority;
});
// Methode mit höchster priority wird standardmäßig gewählt
// User kann per ?id={setupId} eine andere wählen
```

### Form-Flow

```
GET  /multifactor/authentication?id={setupId}
  → Prüft ob $user = getPendingUserChange() (d.h. Login-Phase läuft)
  → createAuthenticationForm() wird aufgerufen
  → Challenge generieren, in Session speichern

POST /multifactor/authentication?id={setupId}
  → Formular-Validierung
  → setup.lock()
  → processAuthenticationForm() in Transaktion
  → applyPendingUserChange() → User ist jetzt eingeloggt
  → UserLoggedIn-Event
  → Redirect zur ursprünglichen URL
```

---

## SessionHandler – MFA-relevante Methoden

**Dateipfad:** `wcfsetup/install/files/lib/system/session/SessionHandler.class.php`

```php
// Nach Login, vor MFA:
// Prüft ob User MFA aktiv hat.
// true  → speichert pendingUserId in Session, gibt true zurück (→ Redirect zu MFA-Form)
// false → loggt User direkt ein, gibt false zurück
public function changeUserAfterMultifactorAuthentication(User $user): bool;

// Nach erfolgreichem MFA:
// Loggt den User aus getPendingUserChange() ein.
// Wirft Exception bei Mismatch.
public function applyPendingUserChange(User $expectedUser): void;

// Gibt den wartenden User zurück (oder null wenn abgelaufen/nicht vorhanden)
public function getPendingUserChange(): ?User;

// Löscht den pendingUserChange-Eintrag
public function clearPendingUserChange(): void;
```

**Session-Lifetime für pending change:** 10 Minuten (Konstante `PENDING_USER_LIFETIME`)

---

## CheckForMultifactorRequirement Middleware

**Dateipfad:** `wcfsetup/install/files/lib/http/middleware/CheckForMultifactorRequirement.class.php`

Diese Middleware erzwingt MFA-Einrichtung für User, die laut Gruppen-Einstellung MFA aktivieren
müssen, es aber noch nicht haben.

```php
// Wird für jede Anfrage ausgeführt:
if ($user->requiresMultifactor() && !$user->multifactorActive) {
    // Redirect zu AccountSecurityPage (außer bei AJAX/Offline-Mode-Seiten)
}
```

**Relevant für das Plugin:** Das Plugin muss sich darum **nicht kümmern**. Sobald ein Credential
über `processManagementForm()` gespeichert wurde und `multifactorActive = 1` gesetzt ist,
greift diese Middleware nicht mehr.

---

## Vollständiger Login-Flow

```
1. User öffnet LoginForm
   ├── Gibt Username + Passwort ein
   └── Submit

2. LoginForm::save()
   ├── UserAuthentication::loginManually() → prüft Passwort
   ├── changeUserAfterMultifactorAuthentication($user)
   │   ├── IF multifactorActive = 1
   │   │   ├── Speichert pendingUserId in Session
   │   │   └── return true
   │   └── IF multifactorActive = 0
   │       ├── changeUser($user) → direkt eingeloggt
   │       └── return false
   └── IF needsMultifactor → Redirect zu MultifactorAuthenticationForm

3. MultifactorAuthenticationForm
   ├── GET: createAuthenticationForm() aufrufen
   │   └── [WebAuthn-Plugin] Challenge generieren + in Session speichern
   └── POST: processAuthenticationForm() aufrufen
       └── [WebAuthn-Plugin] Assertion verifizieren, signCount updaten

4. Nach erfolgreicher MFA:
   ├── applyPendingUserChange() → User eingeloggt
   ├── UserLoggedIn-Event
   └── Redirect zur ursprünglichen URL
```

---

## Eingebaute MFA-Methoden als Referenz

### TOTP (TotpMultifactorMethod)

**Stärken als Referenz:**
- Zeigt wie `FloodControl` korrekt eingesetzt wird
- Zeigt Geräteverwaltung (mehrere Geräte pro User)
- Zeigt `FOR UPDATE` + `minCounter`-Update für Replay-Schutz

**Schwäche für WebAuthn:**
- Nutzt keine async-Operationen – bei WebAuthn ist die Browser-API asynchron

### Email (EmailMultifactorMethod)

**Stärken als Referenz:**
- Zeigt wie einmalige Codes korrekt invalidiert werden
- Einfachste Implementierung des Interfaces

### Backup (BackupMultifactorMethod)

**Stärken als Referenz:**
- Zeigt bcrypt-Passwort-Hashing für gespeicherte Codes
- Zeigt `processManagementForm` mit Rückgabewert ($returnData = Klartext-Codes)
- Zeigt automatische Generierung bei erster MFA-Einrichtung

**Prioritäten (Referenz):**

| Methode | Priority | Wird bevorzugt |
|---------|---------|----------------|
| TOTP | 10 | ✓ (Standard) |
| Email | 5 | Fallback |
| Backup | 1 | Letzter Ausweg |
| **WebAuthn (geplant)** | **20** | **Bevorzugt** |
