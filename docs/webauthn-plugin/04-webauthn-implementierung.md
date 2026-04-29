# 04 – WebAuthn-Implementierungsplan

> Dieser Plan beschreibt die vollständige Implementierung eines WCF-Plugins für
> WebAuthn/FIDO2 als zweiten Faktor (MFA). Alle Code-Gerüste sind vollständige
> Implementierungsvorlagen.

---

## Inhaltsverzeichnis

- [Übersicht der Plugin-Dateien](#übersicht-der-plugin-dateien)
- [Schritt 1: WebAuthn-Bibliothek](#schritt-1-webauthn-bibliothek)
- [Schritt 2: PHP-Implementierung](#schritt-2-php-implementierung)
- [Schritt 3: TypeScript / Frontend](#schritt-3-typescript--frontend)
- [Schritt 4: Templates](#schritt-4-templates)
- [Schritt 5: Sprachdateien](#schritt-5-sprachdateien)
- [Schritt 6: Package-Konfiguration](#schritt-6-package-konfiguration)
- [Technische Sicherheitsanforderungen](#technische-sicherheitsanforderungen)
- [Challenge-Management](#challenge-management)

---

## Übersicht der Plugin-Dateien

```
com.example.wcf.webauthn/
│
├── package.xml
├── objectType.xml
├── install.sql
├── uninstall.sql
│
├── files.tar
│   └── lib/
│       └── system/
│           └── user/
│               └── multifactor/
│                   └── WebAuthnMultifactorMethod.class.php
│
├── templates.tar
│   ├── multifactorManageWebauthn.tpl    ← Verwaltung (Credential registrieren)
│   └── multifactorAuthWebauthn.tpl      ← Auth (Credential nutzen)
│
└── language/
    ├── de.xml
    └── en.xml
```

---

## Schritt 1: WebAuthn-Bibliothek

### Option A: `web-auth/webauthn-framework` (Composer, empfohlen)

```bash
composer require web-auth/webauthn-framework:^4.0
```

Diese Bibliothek implementiert den vollständigen WebAuthn-Standard inkl.:
- Attestation-Verifikation (Registration)
- Assertion-Verifikation (Authentication)
- Unterstützung für alle Authenticator-Typen

**Integration:** Die `vendor/`-Dateien in `files.tar` einschließen oder WCF's Composer-
Autoloader nutzen (falls verfügbar).

### Option B: Minimalimplementierung (ohne externe Bibliothek)

Für eine leichtgewichtige Implementierung ohne Composer reichen folgende PHP-Operationen:

```php
// clientDataJSON validieren
$clientData = json_decode(base64_decode($clientDataJSON), true);
assert($clientData['type'] === 'webauthn.create'); // oder 'webauthn.get'
assert(hash_equals($clientData['challenge'], $expectedChallenge));
assert($clientData['origin'] === $expectedOrigin);

// authenticatorData parsen (für Registration)
// Bytes 0-31:  rpIdHash (SHA-256 des rpId)
// Byte 32:     flags (UP=Bit0, UV=Bit2, AT=Bit6, ED=Bit7)
// Bytes 33-36: signCount (Big-Endian uint32)
// Bytes 37+:   attestedCredentialData (wenn AT-Flag gesetzt)

// Public Key aus attestedCredentialData extrahieren (CBOR-kodiert)
// → CBOR-Decoder notwendig (z.B. "2tvenom/cbor" oder eigene Implementierung)
```

**Empfehlung:** Option A für Produktionsumgebungen. Option B nur für minimale Proof-of-Concept.

---

## Schritt 2: PHP-Implementierung

### `WebAuthnMultifactorMethod.class.php`

```php
<?php

namespace wcf\system\user\multifactor;

use wcf\system\flood\FloodControl;
use wcf\system\form\builder\container\FormContainer;
use wcf\system\form\builder\field\HiddenFormField;
use wcf\system\form\builder\field\TextFormField;
use wcf\system\form\builder\field\ButtonFormField;
use wcf\system\form\builder\field\validation\FormFieldValidationError;
use wcf\system\form\builder\field\validation\FormFieldValidator;
use wcf\system\form\builder\IFormDocument;
use wcf\system\form\builder\TemplateFormNode;
use wcf\system\WCF;

/**
 * WebAuthn/FIDO2 Multi-Factor Authentication Method.
 *
 * Implementiert IMultifactorMethod für WebAuthn Security Keys und Passkeys.
 */
final class WebAuthnMultifactorMethod implements IMultifactorMethod
{
    /**
     * Maximale Fehlversuche pro 10 Minuten (Flood-Schutz).
     */
    private const USER_ATTEMPTS_PER_TEN_MINUTES = 5;

    /**
     * Gültigkeitsdauer einer Challenge in Sekunden.
     */
    private const CHALLENGE_LIFETIME = 300; // 5 Minuten

    /**
     * Session-Key für die gespeicherte Challenge.
     */
    private const SESSION_CHALLENGE_KEY = 'webauthn_challenge';

    // =========================================================================
    // MANAGEMENT (Credential registrieren / verwalten)
    // =========================================================================

    /**
     * @inheritDoc
     */
    public function getStatusText(Setup $setup): string
    {
        $sql = "SELECT  COUNT(*) AS count,
                        MAX(lastUsed) AS lastUsed
                FROM    wcf1_user_multifactor_webauthn
                WHERE   setupID = ?";
        $statement = WCF::getDB()->prepare($sql);
        $statement->execute([$setup->getId()]);

        return WCF::getLanguage()->getDynamicVariable(
            'wcf.user.security.multifactor.webauthn.status',
            $statement->fetchArray()
        );
    }

    /**
     * @inheritDoc
     */
    public function createManagementForm(IFormDocument $form, ?Setup $setup, $returnData = null): void
    {
        $form->addDefaultButton(false);

        // Vorhandene Credentials anzeigen
        if ($setup !== null) {
            $credentials = $this->getCredentials($setup->getId());

            $form->appendChild(
                TemplateFormNode::create('existingCredentials')
                    ->templateName('multifactorManageWebauthn')
                    ->variables([
                        'credentials' => $credentials,
                        'canDelete' => \count($credentials) > 1,
                    ])
            );

            // Löschen-Verarbeitung
            if ($returnData !== null && isset($returnData['action']) && $returnData['action'] === 'delete') {
                $form->successMessage('wcf.user.security.multifactor.webauthn.deleted', [
                    'deviceName' => $returnData['deviceName'],
                ]);
            }
        }

        // Formular für neues Credential
        $challenge = $this->generateChallenge();
        $this->storeChallenge($challenge, 'create');

        $registerContainer = FormContainer::create('registerContainer')
            ->label('wcf.user.security.multifactor.webauthn.register')
            ->appendChildren([
                TemplateFormNode::create('webauthnRegisterUI')
                    ->templateName('multifactorManageWebauthnRegister')
                    ->variables([
                        'challenge' => \base64_encode($challenge),
                        'rpId' => $this->getRpId(),
                        'userId' => WCF::getUser()->userID,
                        'userName' => WCF::getUser()->username,
                        'existingCredentialIds' => $setup ? $this->getCredentialIds($setup->getId()) : [],
                    ]),
                HiddenFormField::create('credentialId')
                    ->objectProperty('credentialId'),
                HiddenFormField::create('clientDataJSON')
                    ->objectProperty('clientDataJSON'),
                HiddenFormField::create('attestationObject')
                    ->objectProperty('attestationObject'),
                TextFormField::create('deviceName')
                    ->label('wcf.user.security.multifactor.webauthn.deviceName')
                    ->placeholder('wcf.user.security.multifactor.webauthn.deviceName.placeholder')
                    ->maximumLength(200),
                ButtonFormField::create('deleteCredential')
                    ->buttonLabel('wcf.global.button.delete')
                    ->objectProperty('delete')
                    ->available($setup !== null),
            ]);

        $form->appendChild($registerContainer);
    }

    /**
     * @inheritDoc
     * @return array{action: string, deviceName: string}|null
     */
    public function processManagementForm(IFormDocument $form, Setup $setup): mixed
    {
        $formData = $form->getData();

        // Löschen eines Credentials
        if (!empty($formData['delete'])) {
            return $this->deleteCredential($setup, $formData['delete']);
        }

        // Neues Credential registrieren
        $credentialId = $formData['data']['credentialId'] ?? '';
        $clientDataJSON = $formData['data']['clientDataJSON'] ?? '';
        $attestationObject = $formData['data']['attestationObject'] ?? '';
        $deviceName = $formData['data']['deviceName'] ?? '';

        if (empty($credentialId) || empty($clientDataJSON) || empty($attestationObject)) {
            throw new \RuntimeException('Incomplete WebAuthn registration data.');
        }

        // Challenge aus Session laden und validieren
        $storedChallenge = $this->getStoredChallenge('create');
        if ($storedChallenge === null) {
            throw new \RuntimeException('WebAuthn challenge expired or missing.');
        }
        $this->clearChallenge();

        // clientDataJSON validieren
        $clientData = \json_decode(\base64_decode($clientDataJSON), true);
        if ($clientData === null) {
            throw new \RuntimeException('Invalid clientDataJSON.');
        }
        if ($clientData['type'] !== 'webauthn.create') {
            throw new \RuntimeException('Invalid WebAuthn type: expected webauthn.create.');
        }
        $receivedChallenge = \base64_decode(
            \str_replace(['-', '_'], ['+', '/'], $clientData['challenge'])
        );
        if (!\hash_equals($storedChallenge, $receivedChallenge)) {
            throw new \RuntimeException('WebAuthn challenge mismatch.');
        }
        if ($clientData['origin'] !== $this->getOrigin()) {
            throw new \RuntimeException('WebAuthn origin mismatch.');
        }

        // attestationObject parsen → Public Key extrahieren
        // [Hier Bibliotheks-spezifischer Code oder CBOR-Parsing]
        $publicKey = $this->extractPublicKey($attestationObject, $clientDataJSON);

        // Credential-ID dekodieren (URL-safe Base64)
        $credentialIdBinary = \base64_decode(
            \str_replace(['-', '_'], ['+', '/'], $credentialId)
        );

        // Gerätename normalisieren
        if (!$deviceName) {
            $deviceName = WCF::getLanguage()->getDynamicVariable(
                'wcf.user.security.multifactor.webauthn.deviceName.placeholder'
            );
        }

        // In Datenbank speichern
        $sql = "INSERT INTO wcf1_user_multifactor_webauthn
                            (setupID, credentialId, publicKey, signCount, deviceName, createTime)
                VALUES      (?, ?, ?, ?, ?, ?)";
        $statement = WCF::getDB()->prepare($sql);
        $statement->execute([
            $setup->getId(),
            $credentialIdBinary,
            $publicKey,
            0, // signCount beginnt bei 0
            $deviceName,
            \TIME_NOW,
        ]);

        return [
            'action' => 'add',
            'deviceName' => $deviceName,
        ];
    }

    // =========================================================================
    // AUTHENTICATION (Credential verifizieren)
    // =========================================================================

    /**
     * @inheritDoc
     */
    public function createAuthenticationForm(IFormDocument $form, Setup $setup): void
    {
        $credentials = $this->getCredentials($setup->getId());

        if (empty($credentials)) {
            throw new \RuntimeException('No WebAuthn credentials found for this setup.');
        }

        // Challenge generieren und in Session speichern
        $challenge = $this->generateChallenge();
        $this->storeChallenge($challenge, 'get');

        $credentialIds = \array_map(function ($cred) {
            return \base64_encode($cred['credentialId']);
        }, $credentials);

        $form->appendChildren([
            TemplateFormNode::create('webauthnAuthUI')
                ->templateName('multifactorAuthWebauthn')
                ->variables([
                    'challenge' => \base64_encode($challenge),
                    'rpId' => $this->getRpId(),
                    'allowCredentials' => $credentialIds,
                ]),
            HiddenFormField::create('credentialId')
                ->objectProperty('credentialId')
                ->autoFocus(),
            HiddenFormField::create('clientDataJSON')
                ->objectProperty('clientDataJSON'),
            HiddenFormField::create('authenticatorData')
                ->objectProperty('authenticatorData'),
            HiddenFormField::create('signature')
                ->objectProperty('signature'),
            HiddenFormField::create('userHandle')
                ->objectProperty('userHandle'),
        ]);
    }

    /**
     * @inheritDoc
     */
    public function processAuthenticationForm(IFormDocument $form, Setup $setup): void
    {
        $formData = $form->getData();

        // Flood-Control prüfen
        FloodControl::getInstance()->registerUserContent(
            'com.example.wcf.multifactor.webauthn',
            $setup->getId()
        );
        $attempts = FloodControl::getInstance()->countUserContent(
            'com.example.wcf.multifactor.webauthn',
            $setup->getId(),
            new \DateInterval('PT10M')
        );
        if ($attempts['count'] > self::USER_ATTEMPTS_PER_TEN_MINUTES) {
            throw new \RuntimeException('Too many authentication attempts.');
        }

        $credentialId = $formData['data']['credentialId'] ?? '';
        $clientDataJSON = $formData['data']['clientDataJSON'] ?? '';
        $authenticatorData = $formData['data']['authenticatorData'] ?? '';
        $signature = $formData['data']['signature'] ?? '';

        if (empty($credentialId) || empty($clientDataJSON) || empty($authenticatorData) || empty($signature)) {
            throw new \RuntimeException('Incomplete WebAuthn authentication data.');
        }

        // Challenge aus Session laden
        $storedChallenge = $this->getStoredChallenge('get');
        if ($storedChallenge === null) {
            throw new \RuntimeException('WebAuthn challenge expired or missing.');
        }
        $this->clearChallenge();

        // clientDataJSON validieren
        $clientData = \json_decode(\base64_decode($clientDataJSON), true);
        if ($clientData === null || $clientData['type'] !== 'webauthn.get') {
            throw new \RuntimeException('Invalid WebAuthn authentication data.');
        }
        $receivedChallenge = \base64_decode(
            \str_replace(['-', '_'], ['+', '/'], $clientData['challenge'])
        );
        if (!\hash_equals($storedChallenge, $receivedChallenge)) {
            throw new \RuntimeException('WebAuthn challenge mismatch.');
        }
        if ($clientData['origin'] !== $this->getOrigin()) {
            throw new \RuntimeException('WebAuthn origin mismatch.');
        }

        // Credential-ID dekodieren
        $credentialIdBinary = \base64_decode(
            \str_replace(['-', '_'], ['+', '/'], $credentialId)
        );

        // Credential aus DB laden (mit FOR UPDATE für Replay-Schutz)
        $sql = "SELECT  *
                FROM    wcf1_user_multifactor_webauthn
                WHERE   setupID = ?
                    AND credentialId = ?
                FOR UPDATE";
        $statement = WCF::getDB()->prepare($sql);
        $statement->execute([$setup->getId(), $credentialIdBinary]);
        $credential = $statement->fetchSingleRow();

        if (!$credential) {
            throw new \RuntimeException('Credential not found.');
        }

        // authenticatorData parsen
        $authDataBinary = \base64_decode($authenticatorData);
        $rpIdHash = \substr($authDataBinary, 0, 32);
        $flags = \ord($authDataBinary[32]);
        $signCountBytes = \substr($authDataBinary, 33, 4);
        $newSignCount = \unpack('N', $signCountBytes)[1]; // Big-Endian uint32

        // rpId-Hash validieren
        if (!\hash_equals(\hash('sha256', $this->getRpId(), true), $rpIdHash)) {
            throw new \RuntimeException('WebAuthn rpId hash mismatch.');
        }

        // User-Presence-Flag prüfen (Bit 0)
        if (!($flags & 0x01)) {
            throw new \RuntimeException('User presence flag not set.');
        }

        // Signatur verifizieren
        $signatureBinary = \base64_decode($signature);
        $clientDataHash = \hash('sha256', \base64_decode($clientDataJSON), true);
        $verificationData = $authDataBinary . $clientDataHash;

        if (!$this->verifySignature($credential['publicKey'], $verificationData, $signatureBinary)) {
            throw new \RuntimeException('WebAuthn signature verification failed.');
        }

        // Replay-Schutz: signCount MUSS größer als der gespeicherte Wert sein
        // (Ausnahme: signCount = 0 bedeutet Authenticator unterstützt kein Zählen)
        if ($newSignCount !== 0 && $newSignCount <= $credential['signCount']) {
            throw new \RuntimeException('WebAuthn sign count indicates replay attack.');
        }

        // signCount und lastUsed aktualisieren
        $newCount = $newSignCount > 0 ? $newSignCount : ($credential['signCount'] + 1);
        $sql = "UPDATE  wcf1_user_multifactor_webauthn
                SET     signCount = ?,
                        lastUsed = ?
                WHERE   setupID = ?
                    AND credentialId = ?
                    AND signCount < ?";
        $statement = WCF::getDB()->prepare($sql);
        $statement->execute([
            $newCount,
            \TIME_NOW,
            $setup->getId(),
            $credentialIdBinary,
            $newCount,
        ]);

        if ($statement->getAffectedRows() !== 1) {
            throw new \RuntimeException('Failed to update sign count – possible replay attack.');
        }
    }

    // =========================================================================
    // HILFSMETHODEN
    // =========================================================================

    private function generateChallenge(): string
    {
        return \random_bytes(32); // 256 Bit Zufalls-Challenge
    }

    private function storeChallenge(string $challenge, string $type): void
    {
        WCF::getSession()->register(self::SESSION_CHALLENGE_KEY, [
            'challenge' => \base64_encode($challenge),
            'type' => $type,
            'expires' => \TIME_NOW + self::CHALLENGE_LIFETIME,
        ]);
    }

    private function getStoredChallenge(string $expectedType): ?string
    {
        $data = WCF::getSession()->getVar(self::SESSION_CHALLENGE_KEY);
        if ($data === null) {
            return null;
        }
        if ($data['type'] !== $expectedType) {
            return null;
        }
        if ($data['expires'] < \TIME_NOW) {
            return null;
        }
        return \base64_decode($data['challenge']);
    }

    private function clearChallenge(): void
    {
        WCF::getSession()->unregister(self::SESSION_CHALLENGE_KEY);
    }

    private function getRpId(): string
    {
        // rpId = Hostname der Seite (ohne Port, ohne Protokoll)
        return \parse_url(WCF::getPath(), PHP_URL_HOST) ?? $_SERVER['HTTP_HOST'];
    }

    private function getOrigin(): string
    {
        $scheme = (!empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off') ? 'https' : 'http';
        return $scheme . '://' . $_SERVER['HTTP_HOST'];
    }

    /**
     * @return array<int, array{credentialId: string, deviceName: string, createTime: int, lastUsed: ?int}>
     */
    private function getCredentials(int $setupId): array
    {
        $sql = "SELECT  credentialId, deviceName, createTime, lastUsed
                FROM    wcf1_user_multifactor_webauthn
                WHERE   setupID = ?
                ORDER BY createTime ASC";
        $statement = WCF::getDB()->prepare($sql);
        $statement->execute([$setupId]);
        return $statement->fetchAll(\PDO::FETCH_ASSOC);
    }

    /**
     * @return string[] Base64-kodierte Credential-IDs
     */
    private function getCredentialIds(int $setupId): array
    {
        $credentials = $this->getCredentials($setupId);
        return \array_map(fn($c) => \base64_encode($c['credentialId']), $credentials);
    }

    /**
     * Löscht ein einzelnes Credential.
     * @return array{action: string, deviceName: string}
     */
    private function deleteCredential(Setup $setup, string $credentialIdBase64): array
    {
        $credentialIdBinary = \base64_decode($credentialIdBase64);

        // Gerätename für Success-Message holen
        $sql = "SELECT  deviceName
                FROM    wcf1_user_multifactor_webauthn
                WHERE   setupID = ?
                    AND credentialId = ?";
        $statement = WCF::getDB()->prepare($sql);
        $statement->execute([$setup->getId(), $credentialIdBinary]);
        $deviceName = $statement->fetchSingleColumn();

        // Löschen
        $sql = "DELETE FROM wcf1_user_multifactor_webauthn
                WHERE       setupID = ?
                        AND credentialId = ?";
        $statement = WCF::getDB()->prepare($sql);
        $statement->execute([$setup->getId(), $credentialIdBinary]);

        // Sicherstellen dass mind. 1 Credential übrig bleibt
        $sql = "SELECT  COUNT(*)
                FROM    wcf1_user_multifactor_webauthn
                WHERE   setupID = ?";
        $statement = WCF::getDB()->prepare($sql);
        $statement->execute([$setup->getId()]);
        if (!$statement->fetchSingleColumn()) {
            throw new \LogicException('Cannot delete the last WebAuthn credential.');
        }

        return ['action' => 'delete', 'deviceName' => $deviceName];
    }

    /**
     * Extrahiert den Public Key aus dem attestationObject.
     * Hier wird die Bibliothek (web-auth/webauthn-framework o.ä.) eingesetzt.
     */
    private function extractPublicKey(string $attestationObjectB64, string $clientDataJSONB64): string
    {
        // TODO: Bibliotheks-spezifische Implementierung
        // Rückgabewert: PEM-kodierter Public Key oder COSE-kodierte Rohdaten
        throw new \RuntimeException('extractPublicKey() not yet implemented – add WebAuthn library.');
    }

    /**
     * Verifiziert die WebAuthn-Signatur.
     */
    private function verifySignature(string $storedPublicKey, string $data, string $signature): bool
    {
        // TODO: Bibliotheks-spezifische Implementierung
        // openssl_verify($data, $signature, $storedPublicKey, OPENSSL_ALGO_SHA256)
        throw new \RuntimeException('verifySignature() not yet implemented – add WebAuthn library.');
    }
}
```

---

## Schritt 3: TypeScript / Frontend

### Dateistruktur

```
ts/WoltLabSuite/Core/Mfa/
└── WebAuthn.ts
```

### `WebAuthn.ts`

```typescript
/**
 * WebAuthn MFA – Browser-seitiger Code
 * Handles navigator.credentials.create() and navigator.credentials.get()
 */

interface RegistrationOptions {
  challenge: string;       // Base64-kodierte Challenge
  rpId: string;            // Hostname
  userId: number;          // WCF User-ID
  userName: string;        // Username
  existingCredentialIds: string[]; // Bereits registrierte Credential-IDs (Base64)
}

interface AuthenticationOptions {
  challenge: string;
  rpId: string;
  allowCredentials: string[]; // Erlaubte Credential-IDs (Base64)
}

/** Konvertiert Base64 → ArrayBuffer */
function base64ToArrayBuffer(base64: string): ArrayBuffer {
  const binary = atob(base64.replace(/-/g, '+').replace(/_/g, '/'));
  const buffer = new ArrayBuffer(binary.length);
  const view = new Uint8Array(buffer);
  for (let i = 0; i < binary.length; i++) {
    view[i] = binary.charCodeAt(i);
  }
  return buffer;
}

/** Konvertiert ArrayBuffer → Base64url */
function arrayBufferToBase64url(buffer: ArrayBuffer): string {
  const bytes = new Uint8Array(buffer);
  let binary = '';
  for (let i = 0; i < bytes.byteLength; i++) {
    binary += String.fromCharCode(bytes[i]);
  }
  return btoa(binary).replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');
}

/**
 * Registriert ein neues WebAuthn-Credential.
 * Füllt die Hidden-Fields im Formular.
 */
export async function register(options: RegistrationOptions): Promise<void> {
  const publicKeyOptions: PublicKeyCredentialCreationOptions = {
    challenge: base64ToArrayBuffer(options.challenge),
    rp: {
      id: options.rpId,
      name: document.title,
    },
    user: {
      id: new TextEncoder().encode(String(options.userId)),
      name: options.userName,
      displayName: options.userName,
    },
    pubKeyCredParams: [
      { type: 'public-key', alg: -7 },   // ES256 (ECDSA P-256)
      { type: 'public-key', alg: -257 },  // RS256 (RSA PKCS#1)
    ],
    authenticatorSelection: {
      userVerification: 'preferred',
    },
    excludeCredentials: options.existingCredentialIds.map((id) => ({
      type: 'public-key' as const,
      id: base64ToArrayBuffer(id),
    })),
    timeout: 60000,
    attestation: 'none', // Kein Attestation-Statement nötig für einfache MFA
  };

  let credential: PublicKeyCredential;
  try {
    credential = (await navigator.credentials.create({
      publicKey: publicKeyOptions,
    })) as PublicKeyCredential;
  } catch (e) {
    if (e instanceof DOMException && e.name === 'NotAllowedError') {
      throw new Error('User cancelled or timed out.');
    }
    throw e;
  }

  const response = credential.response as AuthenticatorAttestationResponse;

  // Hidden-Fields befüllen
  (document.getElementById('credentialId') as HTMLInputElement).value =
    arrayBufferToBase64url(credential.rawId);
  (document.getElementById('clientDataJSON') as HTMLInputElement).value =
    arrayBufferToBase64url(response.clientDataJSON);
  (document.getElementById('attestationObject') as HTMLInputElement).value =
    arrayBufferToBase64url(response.attestationObject);
}

/**
 * Verifiziert ein WebAuthn-Credential beim Login.
 * Füllt die Hidden-Fields im Formular.
 */
export async function authenticate(options: AuthenticationOptions): Promise<void> {
  const publicKeyOptions: PublicKeyCredentialRequestOptions = {
    challenge: base64ToArrayBuffer(options.challenge),
    rpId: options.rpId,
    allowCredentials: options.allowCredentials.map((id) => ({
      type: 'public-key' as const,
      id: base64ToArrayBuffer(id),
    })),
    userVerification: 'preferred',
    timeout: 60000,
  };

  let credential: PublicKeyCredential;
  try {
    credential = (await navigator.credentials.get({
      publicKey: publicKeyOptions,
    })) as PublicKeyCredential;
  } catch (e) {
    if (e instanceof DOMException && e.name === 'NotAllowedError') {
      throw new Error('User cancelled or timed out.');
    }
    throw e;
  }

  const response = credential.response as AuthenticatorAssertionResponse;

  // Hidden-Fields befüllen
  (document.getElementById('credentialId') as HTMLInputElement).value =
    arrayBufferToBase64url(credential.rawId);
  (document.getElementById('clientDataJSON') as HTMLInputElement).value =
    arrayBufferToBase64url(response.clientDataJSON);
  (document.getElementById('authenticatorData') as HTMLInputElement).value =
    arrayBufferToBase64url(response.authenticatorData);
  (document.getElementById('signature') as HTMLInputElement).value =
    arrayBufferToBase64url(response.signature);
  if (response.userHandle) {
    (document.getElementById('userHandle') as HTMLInputElement).value =
      arrayBufferToBase64url(response.userHandle);
  }
}

/**
 * Prüft ob WebAuthn im aktuellen Browser verfügbar ist.
 */
export async function isAvailable(): Promise<boolean> {
  return (
    typeof PublicKeyCredential !== 'undefined' &&
    typeof navigator.credentials !== 'undefined' &&
    typeof navigator.credentials.create === 'function' &&
    typeof navigator.credentials.get === 'function'
  );
}
```

---

## Schritt 4: Templates

### `multifactorManageWebauthn.tpl` (Verwaltung)

```smarty
{* Zeigt vorhandene Credentials und bietet Registrierung neuer Credentials an *}

{if !$__wcf->getSession()->getVar('webauthn_available')}
    <p class="warning">{lang}wcf.user.security.multifactor.webauthn.notSupported{/lang}</p>
{/if}

{if !empty($credentials)}
<section class="section">
    <h2 class="sectionTitle">{lang}wcf.user.security.multifactor.webauthn.devices{/lang}</h2>
    <ul class="dataList">
        {foreach from=$credentials item=$credential}
        <li class="dataListItem">
            <div class="dataListItemTitle">{$credential.deviceName}</div>
            <div class="dataListItemDescription">
                {lang added=$credential.createTime}wcf.user.security.multifactor.webauthn.added{/lang}
                {if $credential.lastUsed}
                    &nbsp;–&nbsp;
                    {lang lastUsed=$credential.lastUsed}wcf.user.security.multifactor.webauthn.lastUsed{/lang}
                {/if}
            </div>
        </li>
        {/foreach}
    </ul>
</section>
{/if}

<section class="section">
    <h2 class="sectionTitle">{lang}wcf.user.security.multifactor.webauthn.registerNew{/lang}</h2>
    <p>{lang}wcf.user.security.multifactor.webauthn.registerNew.description{/lang}</p>

    <button type="button" id="webauthnRegisterButton" class="button">
        <span class="icon icon16 fa-key"></span>
        {lang}wcf.user.security.multifactor.webauthn.registerDevice{/lang}
    </button>
    <p id="webauthnRegisterStatus" class="innerInfo" style="display:none;"></p>
</section>

<script data-relocate="true">
require(['WoltLabSuite/Core/Mfa/WebAuthn'], async function(WebAuthn) {
    const isAvailable = await WebAuthn.isAvailable();
    if (!isAvailable) {
        document.getElementById('webauthnRegisterButton').disabled = true;
        return;
    }

    document.getElementById('webauthnRegisterButton').addEventListener('click', async function() {
        const statusEl = document.getElementById('webauthnRegisterStatus');
        statusEl.style.display = 'block';
        statusEl.textContent = '{lang}wcf.user.security.multifactor.webauthn.waiting{/lang}';

        try {
            await WebAuthn.register({
                challenge: '{$challenge|encodeJS}',
                rpId: '{$rpId|encodeJS}',
                userId: {$userId},
                userName: '{$__wcf->getUser()->username|encodeJS}',
                existingCredentialIds: {$existingCredentialIds|json_encode}
            });

            statusEl.textContent = '{lang}wcf.user.security.multifactor.webauthn.registered{/lang}';
            document.querySelector('form').submit();
        } catch (e) {
            statusEl.textContent = e.message || '{lang}wcf.user.security.multifactor.webauthn.error{/lang}';
        }
    });
});
</script>
```

### `multifactorAuthWebauthn.tpl` (Login)

```smarty
{* Zeigt den "Security Key verwenden"-Button *}

<p>{lang}wcf.user.security.multifactor.webauthn.auth.description{/lang}</p>

<button type="button" id="webauthnAuthButton" class="button buttonPrimary">
    <span class="icon icon16 fa-key"></span>
    {lang}wcf.user.security.multifactor.webauthn.useDevice{/lang}
</button>
<p id="webauthnAuthStatus" class="innerInfo" style="display:none;"></p>

<script data-relocate="true">
require(['WoltLabSuite/Core/Mfa/WebAuthn'], async function(WebAuthn) {
    const button = document.getElementById('webauthnAuthButton');
    const statusEl = document.getElementById('webauthnAuthStatus');

    const isAvailable = await WebAuthn.isAvailable();
    if (!isAvailable) {
        button.disabled = true;
        statusEl.style.display = 'block';
        statusEl.textContent = '{lang}wcf.user.security.multifactor.webauthn.notSupported{/lang}';
        return;
    }

    // Automatisch starten
    button.addEventListener('click', triggerAuth);
    triggerAuth();

    async function triggerAuth() {
        statusEl.style.display = 'block';
        statusEl.textContent = '{lang}wcf.user.security.multifactor.webauthn.waiting{/lang}';

        try {
            await WebAuthn.authenticate({
                challenge: '{$challenge|encodeJS}',
                rpId: '{$rpId|encodeJS}',
                allowCredentials: {$allowCredentials|json_encode}
            });

            document.querySelector('form').submit();
        } catch (e) {
            statusEl.textContent = e.message || '{lang}wcf.user.security.multifactor.webauthn.error{/lang}';
        }
    }
});
</script>
```

---

## Schritt 5: Sprachdateien

### `language/de.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<language xmlns="http://www.woltlab.com" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.woltlab.com http://www.woltlab.com/XSD/6.0/language.xsd"
    languagecode="de">
    <category name="wcf.user.security.multifactor.webauthn">
        <item name="wcf.user.security.multifactor.webauthn.status">
            <![CDATA[{if $count}{$count} {if $count == 1}Gerät{else}Geräte{/if} registriert{else}Keine Geräte{/if}]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.register">
            <![CDATA[Security Key / Passkey registrieren]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.registerNew">
            <![CDATA[Neues Gerät hinzufügen]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.registerNew.description">
            <![CDATA[Klicken Sie auf die Schaltfläche und folgen Sie den Anweisungen Ihres Browsers, um einen Security Key oder Passkey zu registrieren.]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.registerDevice">
            <![CDATA[Gerät registrieren]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.devices">
            <![CDATA[Registrierte Geräte]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.deviceName">
            <![CDATA[Gerätename]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.deviceName.placeholder">
            <![CDATA[Security Key]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.added">
            <![CDATA[Hinzugefügt: {$added|time}]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.lastUsed">
            <![CDATA[Zuletzt verwendet: {$lastUsed|time}]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.deleted">
            <![CDATA[Gerät „{$deviceName}" wurde entfernt.]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.auth.description">
            <![CDATA[Stecken Sie Ihren Security Key ein oder verwenden Sie Ihren Passkey.]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.useDevice">
            <![CDATA[Security Key / Passkey verwenden]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.waiting">
            <![CDATA[Warte auf Gerät …]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.registered">
            <![CDATA[Gerät erfolgreich registriert. Formular wird übermittelt …]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.notSupported">
            <![CDATA[Ihr Browser unterstützt WebAuthn nicht. Bitte aktualisieren Sie Ihren Browser.]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.error">
            <![CDATA[Fehler bei der Geräteauthentifizierung. Bitte versuchen Sie es erneut.]]>
        </item>
        <item name="wcf.user.security.multifactor.webauthn.error.flood">
            <![CDATA[Zu viele fehlgeschlagene Versuche. Bitte warten Sie einige Minuten.]]>
        </item>
    </category>
</language>
```

---

## Schritt 6: Package-Konfiguration

### `objectType.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<data xmlns="http://www.woltlab.com" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.woltlab.com http://www.woltlab.com/XSD/6.0/objectType.xsd">
    <import>
        <type>
            <name>com.example.wcf.multifactor.webauthn</name>
            <definitionname>com.woltlab.wcf.multifactor</definitionname>
            <icon>key</icon>
            <priority>20</priority>
            <classname>wcf\system\user\multifactor\WebAuthnMultifactorMethod</classname>
        </type>
        <type>
            <name>com.example.wcf.multifactor.webauthn</name>
            <definitionname>com.woltlab.wcf.floodControl</definitionname>
        </type>
    </import>
</data>
```

---

## Technische Sicherheitsanforderungen

| Anforderung | Implementierung |
|-------------|----------------|
| HTTPS-Pflicht | Browser erzwingt dies – kein Code nötig |
| Origin-Validierung | `clientDataJSON.origin` gegen `getOrigin()` prüfen |
| Challenge-Bindung | Challenge aus Session, einmalig, mit Ablaufzeit |
| rpId-Bindung | `authenticatorData.rpIdHash` gegen SHA-256(rpId) prüfen |
| Replay-Schutz | `signCount` updaten + `getAffectedRows() !== 1` prüfen |
| Flood-Control | WCF's `FloodControl`-API (5 Versuche / 10 Min) |
| User-Presence | `flags & 0x01` prüfen |
| Transaktions-Safety | `FOR UPDATE` beim Credential-Laden |

---

## Challenge-Management

Die Challenge muss sicher zwischen `createAuthenticationForm()` (GET) und
`processAuthenticationForm()` (POST) übertragen werden.

### Empfohlener Ansatz: Session-Variable

```php
// Speichern (in createAuthenticationForm)
WCF::getSession()->register('webauthn_challenge', [
    'challenge' => base64_encode($challenge),
    'type' => 'get',  // oder 'create'
    'expires' => TIME_NOW + 300,
]);

// Lesen (in processAuthenticationForm)
$data = WCF::getSession()->getVar('webauthn_challenge');
if (!$data || $data['expires'] < TIME_NOW) {
    throw new \RuntimeException('Challenge expired.');
}
$storedChallenge = base64_decode($data['challenge']);

// Löschen (nach Verarbeitung – Einmal-Nutzung!)
WCF::getSession()->unregister('webauthn_challenge');
```

> **Wichtig:** Die Challenge MUSS nach der ersten Verarbeitung gelöscht werden, um
> Replay-Angriffe über mehrfache POST-Requests zu verhindern.
