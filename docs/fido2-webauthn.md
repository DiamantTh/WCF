# FIDO2 / WebAuthn (Passkeys)

WoltLab Suite Core 6.x supports passkey-based login (FIDO2 / WebAuthn). This document explains how the feature works internally and how your package can integrate with or extend the authentication layer.

---

## 1. Background

**FIDO2** is the umbrella term for the W3C WebAuthn specification combined with the FIDO Alliance's CTAP2 protocol. From a developer's perspective the relevant standard is [WebAuthn (W3C)](https://www.w3.org/TR/webauthn-3/).

A *passkey* is a FIDO2 credential stored on a device (hardware security key, platform authenticator such as Windows Hello / Face ID, or a password manager). The browser exposes it via the `navigator.credentials` API.

---

## 2. Server-side libraries

WCS bundles the following PHP libraries to handle WebAuthn:

| Library | Purpose |
|---|---|
| `web-token/jwt-library` | JSON Web Token handling (used for challenge signing) |
| `spomky-labs/pki-framework` | ASN.1 / X.509 parsing used in attestation |

These are located under `wcfsetup/install/files/lib/system/api/`.

---

## 3. Authentication Pipeline

WCS uses an **authentication pipeline** architecture. Each authentication mechanism (password, TOTP, passkey, …) is registered as an `objectType` against the definition `com.woltlab.wcf.user.authentication.webAuthn` (or the broader `com.woltlab.wcf.user.loginHandler`).

### Flow

```
Browser                          WCS Server
  |                                  |
  |-- POST /login (username) ------->|
  |                                  |-- lookup user
  |<-- challenge (JSON) -------------|
  |                                  |
  |-- WebAuthn assertion ----------->|  (navigator.credentials.get)
  |                                  |-- verify signature
  |<-- session cookie ---------------|
```

### Key PHP classes

| Class | Role |
|---|---|
| `wcf\system\user\authentication\UserAuthenticationFactory` | Resolves the active authentication handler |
| `wcf\system\user\authentication\DefaultUserAuthentication` | Password-based authentication (reference implementation) |

---

## 4. Registering a Custom Authentication Handler

To add a custom second-factor or alternative login method, register an object type:

```xml
<!-- objectType.xml -->
<data xmlns="http://www.woltlab.com" …>
    <import>
        <type>
            <name>com.example.mypkg.user.authentication.myMethod</name>
            <definitionname>com.woltlab.wcf.user.loginHandler</definitionname>
            <classname>com\example\mypkg\system\user\authentication\MyMethodUserAuthentication</classname>
        </type>
    </import>
</data>
```

Your class must implement `wcf\system\user\authentication\IUserAuthentication` (or the relevant interface for your handler type).

---

## 5. JavaScript / Browser side

WCS ships TypeScript source under `ts/`. The relevant WebAuthn helper is:

```
ts/WoltLabSuite/Core/WebAuthn/
```

Use the exported functions to interact with `navigator.credentials` and communicate with the server-side endpoints.

### Example: triggering a passkey assertion

```typescript
import { getAssertion } from "WoltLabSuite/Core/WebAuthn/Assert";

const result = await getAssertion(challenge);
// Send `result` to your server endpoint for verification
```

---

## 6. Endpoints

WebAuthn endpoints are regular WCS controller actions (`AbstractSecureAction` subclasses) registered via the `page` PIP or invoked as AJAX actions. They follow the standard WCS request/response cycle.

---

## 7. Security Considerations

* Always validate the **origin** and **rpId** in attestation/assertion responses on the server side.
* Store only the **public key** and **credential ID**, never the private key.
* Implement **rate limiting** on challenge endpoints to prevent enumeration attacks.
* Use `challenge` values with at least 128 bits of entropy (WCS uses `random_bytes(32)`).
* Set `userVerification: "required"` for high-security flows (account settings changes, payment, etc.).

---

## 8. Further Reading

* [W3C WebAuthn Level 3](https://www.w3.org/TR/webauthn-3/)
* [FIDO Alliance overview](https://fidoalliance.org/passkeys/)
* [passkeys.dev](https://passkeys.dev/) – implementation guides and browser compatibility tables
* [WoltLab Developer Docs](https://docs.woltlab.com/) – official WCS developer documentation
