# WoltLab Suite Core – Developer Documentation

This folder contains developer documentation for **WoltLab Suite Core (WCF/WCS)**.

## Contents

### General documentation

| Document | Description |
|---|---|
| [Plugin API](plugin-api.md) | Event system, Package Installation Plugins (PIPs), and the object-type extension system |
| [FIDO2 / WebAuthn](fido2-webauthn.md) | Passkey/WebAuthn authentication integration and the authentication API |
| [Module System](module-system.md) | Building re-usable modules (e.g. a forum or file database) on top of WCS |
| [Package Development](package-development.md) | Step-by-step guide to creating, building, and installing a WCS package |

### WebAuthn plugin – in-depth documentation (German)

The `webauthn-plugin/` sub-folder contains the detailed analysis and implementation documentation that was originally tracked on the `copilot/conduct-read-only-analysis` branch. It covers everything needed to build WebAuthn/FIDO2 as a standalone WCS plugin.

| Document | Description |
|---|---|
| [webauthn-plugin/README.md](webauthn-plugin/README.md) | Entry point and overview of the WebAuthn plugin docs |
| [01 – Motivation & Kontext](webauthn-plugin/01-motivation-kontext.md) | Why a plugin rather than a core change? |
| [02 – WCF Plugin-API](webauthn-plugin/02-plugin-api.md) | Complete analysis of all relevant WCF plugin APIs |
| [03 – MFA-Subsystem](webauthn-plugin/03-mfa-subsystem.md) | Deep-dive into the WCF multi-factor authentication subsystem |
| [04 – WebAuthn-Implementierung](webauthn-plugin/04-webauthn-implementierung.md) | Implementation plan and code scaffolding |
| [05 – Datenbankschema](webauthn-plugin/05-datenbankschema.md) | Database schema for credential storage |
| [06 – Migrationsstrategie](webauthn-plugin/06-migration-strategie.md) | Migration path from plugin to core |
| [07 – Plugin-API allgemein](webauthn-plugin/07-plugin-api-allgemein.md) | General plugin API overview and plugin ideas catalogue |

## Quick-start

1. Read **[Package Development](package-development.md)** to understand the fundamental packaging mechanism.
2. Read **[Plugin API](plugin-api.md)** to learn how to extend core functionality without forking the source.
3. Read **[Module System](module-system.md)** if you want to build a self-contained application module (e.g. a forum).
4. Read **[FIDO2 / WebAuthn](fido2-webauthn.md)** for a concise introduction to passkey authentication in WCS.
5. Read **[webauthn-plugin/](webauthn-plugin/README.md)** for the full in-depth WebAuthn plugin analysis and implementation guide.

## About this branch

This branch (`copilot/general-module-system`) is the consolidated general development and documentation branch. It supersedes the earlier `copilot/conduct-read-only-analysis` branch (the original WebAuthn analysis branch), whose entire documentation has been carried over to `docs/webauthn-plugin/`.
