# WoltLab Suite Core – Developer Documentation

This folder contains developer documentation for **WoltLab Suite Core (WCF/WCS)**.

## Contents

| Document | Description |
|---|---|
| [Plugin API](plugin-api.md) | Event system, Package Installation Plugins (PIPs), and the object-type extension system |
| [FIDO2 / WebAuthn](fido2-webauthn.md) | Passkey/WebAuthn authentication integration and the authentication API |
| [Module System](module-system.md) | Building re-usable modules (e.g. a forum or file database) on top of WCS |
| [Package Development](package-development.md) | Step-by-step guide to creating, building, and installing a WCS package |

## Quick-start

1. Read **[Package Development](package-development.md)** to understand the fundamental packaging mechanism.
2. Read **[Plugin API](plugin-api.md)** to learn how to extend core functionality without forking the source.
3. Read **[Module System](module-system.md)** if you want to build a self-contained application module (e.g. a forum).
4. Read **[FIDO2 / WebAuthn](fido2-webauthn.md)** if you need passkey / strong authentication support in your package.

## About this branch

This branch (`copilot/general-module-system`) serves as the general development and documentation branch, consolidating topics that were previously tracked separately (including the earlier WebAuthn / FIDO2 work).
