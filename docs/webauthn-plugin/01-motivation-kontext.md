# 01 – Motivation & Kontext

## Stand der Dinge (März 2026)

WebAuthn/FIDO2/Passkeys ist in WoltLab Suite Core **noch nicht im offiziellen Core** enthalten.
Der Hintergrund: WoltLab bietet WebAuthn derzeit als **kostenpflichtiges Plugin** für das WoltLab
Community Framework (WCF/WSC) an – das war schon vor der 6.x-Serie so. Der Core-Einbau wird zwar
intern diskutiert, hat aber keine konkrete Zeitplanung.

### Warum ein eigenes Plugin?

| Faktor | Plugin-Weg | Core-Weg |
|--------|-----------|----------|
| Zeitrahmen | Sofort umsetzbar | Unbekannt / nicht absehbar |
| Abhängigkeiten | Nur `com.woltlab.wcf` ≥ 5.4 | Fork / PR ins offizielle Repo |
| Risiko | Gering – nutzt stabiles Interface | Hoch – Kernänderungen |
| Migrations-Fähigkeit | Ja – Datenmigration planbar | – |
| Wartung | Eigenverantwortlich | Im Core-Team |

### Warum ist Migration später möglich?

Das MFA-Subsystem speichert Credential-Daten **pro `setupID`**, die aus der zentralen
`wcf1_user_multifactor`-Tabelle stammt. Die eigene Plugin-Tabelle (z.B.
`wcf1_user_multifactor_webauthn`) lässt sich bei einem späteren Core-Einbau mit einem
Datenbankskript in das offizielle Schema überführen – sofern man beim Datenbankdesign
kompatibel bleibt.

## WebAuthn-Grundlagen (technische Kurzreferenz)

### Was ist WebAuthn?

**WebAuthn** (Web Authentication API, W3C-Standard) ist ein Browser-Standard, der Hardware-
und Software-Authenticatoren (Security Keys, Biometrie, Passkeys) für die sichere
Authentifizierung nutzbar macht. Es ist der Web-Teil des FIDO2-Standards.

```
FIDO2
├── WebAuthn (Browser ↔ Relying Party / Server-API)
└── CTAP2   (Browser ↔ Authenticator-Hardware/Software)
```

### Zwei Operationen

#### 1. Registration (Credential anlegen)

```
Server                                    Browser/Authenticator
  │                                              │
  ├─── PublicKeyCredentialCreationOptions ──────►│
  │    (challenge, rp, user, pubKeyParams)       │
  │                                              │
  │◄─── AuthenticatorAttestationResponse ────────┤
  │    (id, rawId, attestationObject,            │
  │     clientDataJSON)                          │
  │                                              │
  ├─── Validierung & Speicherung ──────────────►  │
```

#### 2. Authentication (Assertion prüfen)

```
Server                                    Browser/Authenticator
  │                                              │
  ├─── PublicKeyCredentialRequestOptions ───────►│
  │    (challenge, allowCredentials, rpId)       │
  │                                              │
  │◄─── AuthenticatorAssertionResponse ──────────┤
  │    (id, rawId, authenticatorData,            │
  │     clientDataJSON, signature)               │
  │                                              │
  ├─── Validierung (sig, counter, origin) ──────►  │
```

### Sicherheitsrelevante Daten

| Feld | Beschreibung | Wo gespeichert |
|------|-------------|----------------|
| `credentialId` | Eindeutige ID des Credentials | Server (DB) |
| `publicKey` | COSE-kodierter Public Key | Server (DB) |
| `signCount` | Replay-Schutz (Zähler) | Server (DB), wird pro Login aktualisiert |
| `aaguid` | Authenticator-Typ-ID | Server (DB, optional) |
| `userHandle` | Verknüpfung zu User | Browser (optional gespeichert) |
| `privateKey` | Geheimschlüssel | **Nur im Authenticator** – verlässt diesen nie |

### Passkeys vs. Security Keys

| Typ | Beschreibung | Beispiele |
|-----|-------------|----------|
| **Roaming Authenticator** | Externes Gerät, per USB/NFC/BLE | YubiKey, Nitrokey |
| **Platform Authenticator** | Geräteintern (Biometrie, PIN) | Touch ID, Windows Hello, Android |
| **Passkey (Synced)** | Cloud-synchronisierter Platform-Authenticator | iCloud Keychain, Google Password Manager |

Im Plugin-Kontext: Alles davon funktioniert – der Server unterscheidet nicht, ob es ein
Hardware-Key oder ein Passkey ist. Der Unterschied liegt nur im `authenticatorAttachment`-Hint.

## Systemvoraussetzungen für das Plugin

- WoltLab Suite Core ≥ 5.4 (MFA-Subsystem eingeführt)
- PHP ≥ 8.1
- PHP-Extension: `openssl` (für Signaturprüfung)
- PHP-Bibliothek: `web-auth/webauthn-framework` ≥ 4.x (via Composer) **oder** eigene Implementierung
- Browser: Alle modernen Browser (Chrome 67+, Firefox 60+, Safari 14+, Edge 18+)
- HTTPS: **Pflicht** – WebAuthn funktioniert nur über HTTPS (oder `localhost`)
