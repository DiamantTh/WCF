# WebAuthn für WoltLab Suite Core – Plugin-Dokumentation

> **Zielgruppe:** Entwickler, die WebAuthn/FIDO2/Passkeys als zweiten Faktor (MFA)
> in WoltLab Suite Core (WCF) als eigenständiges Plugin implementieren wollen.
>
> **WCF-Version:** 6.2.x (getestet gegen Quellstand März 2026)

---

## Inhaltsverzeichnis

1. [Motivation & Kontext](01-motivation-kontext.md)
2. [WCF Plugin-API – Vollständige Analyse](02-plugin-api.md)
3. [MFA-Subsystem – Technische Tiefenanalyse](03-mfa-subsystem.md)
4. [WebAuthn-Implementierungsplan](04-webauthn-implementierung.md)
5. [Datenbankschema](05-datenbankschema.md)
6. [Migrations-Strategie (Plugin → Core)](06-migration-strategie.md)

---

## Kurzübersicht

```
WCF 6.2 MFA-Architektur (Plugin-Erweiterungspunkt)

┌─────────────────────────────────────────────────────────┐
│                   LoginForm (ACP/Frontend)                │
│   validatePassword() → changeUserAfterMultifactor()      │
└────────────────────────┬────────────────────────────────┘
                         │ needsMultifactor = true
                         ▼
┌─────────────────────────────────────────────────────────┐
│              MultifactorAuthenticationForm               │
│   Lädt alle Setup-Instanzen des Users, sortiert nach    │
│   objectType.priority DESC                               │
└────────────────────────┬────────────────────────────────┘
                         │ ruft auf:
                         ▼
┌─────────────────────────────────────────────────────────┐
│              IMultifactorMethod (Interface)              │
│  ← HIER wird das WebAuthn-Plugin eingehängt             │
│                                                          │
│  + createAuthenticationForm()  (Challenge anzeigen)     │
│  + processAuthenticationForm() (Assertion prüfen)       │
│  + createManagementForm()      (Key registrieren)       │
│  + processManagementForm()     (Key speichern)          │
│  + getStatusText()             (Status-Anzeige)         │
└─────────────────────────────────────────────────────────┘
```

---

## Repository-Struktur dieser Dokumentation

```
docs/webauthn-plugin/
├── README.md                        ← diese Datei (Einstieg)
├── 01-motivation-kontext.md         ← Warum Plugin statt Core?
├── 02-plugin-api.md                 ← Vollständige WCF Plugin-API-Analyse
├── 03-mfa-subsystem.md              ← MFA-Subsystem im Detail
├── 04-webauthn-implementierung.md   ← Implementierungsplan & Code-Gerüst
├── 05-datenbankschema.md            ← SQL-Schema für Credentials
└── 06-migration-strategie.md        ← Weg von Plugin zu Core
```
