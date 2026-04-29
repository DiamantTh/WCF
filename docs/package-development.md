# Package Development

This guide explains how to create, build, and install a package for WoltLab Suite Core (WCS).

---

## 1. Concepts

A WCS **package** is a `.tar` (or `.tar.gz`) archive that contains:

* `package.xml` – the package manifest (metadata, dependencies, instructions)
* One or more data archives (`.tar` files) and/or PHP scripts referenced by `package.xml`
* Optionally: language XML files, templates, database migration scripts, etc.

The package manager unpacks the archive, reads `package.xml`, and executes each `<instruction>` in order.

---

## 2. `package.xml` Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<package name="com.example.mypkg"
         xmlns="http://www.woltlab.com"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://www.woltlab.com http://www.woltlab.com/XSD/6.0/package.xsd">

    <packageinformation>
        <packagename>My Package</packagename>
        <packagename language="de">Mein Paket</packagename>
        <packagedescription>A short description.</packagedescription>
        <version>1.0.0</version>
        <date>2026-01-01</date>
        <!-- Set to 1 only for self-contained applications (e.g. a forum module) -->
        <!-- <isapplication>1</isapplication> -->
        <!-- <applicationdirectory>mypkg</applicationdirectory> -->
    </packageinformation>

    <authorinformation>
        <author>Your Name</author>
        <authorurl>https://example.com</authorurl>
    </authorinformation>

    <!-- Packages that must be installed before this one -->
    <requiredpackages>
        <requiredpackage minversion="6.0.0">com.woltlab.wcf</requiredpackage>
    </requiredpackages>

    <!-- Packages that must NOT be installed at the same time -->
    <!-- <excludedpackages>
        <excludedpackage version="2.0.0 Alpha 1">com.example.conflicting</excludedpackage>
    </excludedpackages> -->

    <instructions type="install">
        <!-- Files (PHP, JS, CSS, …) -->
        <instruction type="file" />

        <!-- Smarty templates -->
        <instruction type="template" />

        <!-- ACP templates -->
        <instruction type="acpTemplate" />

        <!-- Language strings -->
        <instruction type="language" />

        <!-- Database schema migration -->
        <instruction type="database">acp/install_com.example.mypkg.php</instruction>

        <!-- Event listeners -->
        <instruction type="eventListener" />

        <!-- Object types -->
        <instruction type="objectType" />

        <!-- Pages -->
        <instruction type="page" />

        <!-- ACP navigation -->
        <instruction type="acpMenu" />

        <!-- Per-group permissions -->
        <instruction type="userGroupOption" />

        <!-- Cron jobs -->
        <instruction type="cronjob" />
    </instructions>

    <!-- Update instructions (one block per supported upgrade path) -->
    <instructions type="update" fromversion="1.0.0">
        <instruction type="database">acp/update_com.example.mypkg_1.0.1.php</instruction>
        <instruction type="language" />
    </instructions>

</package>
```

### Important attributes

| Field | Notes |
|---|---|
| `name` | Reverse-domain package identifier (must be globally unique) |
| `version` | Semver-style: `Major.Minor.Patch` or `Major.Minor.Patch Alpha/Beta/RC N` |
| `date` | `YYYY-MM-DD` – must be today or in the past |
| `isapplication` | Set to `1` to create a standalone application (installs into its own subdirectory) |
| `applicationdirectory` | Directory name for the application (only if `isapplication=1`) |

---

## 3. Directory Layout of the Source Tree

```
com.example.mypkg/           ← package root
├── package.xml
├── files.tar                ← packed by build script from files/
├── templates.tar            ← packed from templates/
├── acpTemplates.tar         ← packed from acpTemplates/
├── language/
│   ├── de.xml
│   └── en.xml
├── objectType.xml
├── eventListener.xml
├── page.xml
├── acpMenu.xml
├── userGroupOption.xml
├── cronjob.xml
└── acp/
    └── install_com.example.mypkg.php
```

### `files/` layout

The directory structure under `files/` is copied verbatim to the application directory:

```
files/
└── lib/
    ├── action/
    │   └── MyAction.class.php
    ├── form/
    │   └── MyForm.class.php
    ├── page/
    │   └── MyPage.class.php
    └── system/
        └── …
```

### Template files

```
templates/
└── myPage.tpl

acpTemplates/
└── myAcpPage.tpl
```

---

## 4. Language Files

Language XML files are installed by the `language` PIP.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<language xmlns="http://www.woltlab.com" languagecode="en">
    <category name="mymodule.global">
        <item name="mymodule.global.title">My Module</item>
    </category>
    <category name="mymodule.page.index">
        <item name="mymodule.page.index.title">Overview</item>
    </category>
</language>
```

Use `WCF::getLanguage()->get('mymodule.global.title')` in PHP or `{lang}mymodule.global.title{/lang}` in templates.

---

## 5. Building the Package Archive

A minimal build script (Bash):

```bash
#!/usr/bin/env bash
set -euo pipefail

PKG="com.example.mypkg"
VERSION="1.0.0"
OUT="${PKG}_${VERSION}.tar"

# Pack files
tar -cf files.tar -C files .
tar -cf templates.tar -C templates .
tar -cf acpTemplates.tar -C acpTemplates .

# Pack the top-level archive
tar -cf "${OUT}" \
    package.xml \
    files.tar \
    templates.tar \
    acpTemplates.tar \
    language/ \
    objectType.xml \
    eventListener.xml \
    page.xml \
    acpMenu.xml \
    userGroupOption.xml \
    acp/

echo "Built ${OUT}"
```

> **Tip**: The WoltLab plugin-store and community tools provide more sophisticated build toolchains. The above is intentionally minimal to illustrate the mechanism.

---

## 6. Installing the Package

1. Log in to the **ACP** (Administration Control Panel) of your WCS installation.
2. Navigate to **Configuration → Packages → Install Package**.
3. Upload the `.tar` file.
4. Follow the installation wizard.

Alternatively, place the archive in the `packages/` update-server directory and configure a local update server.

---

## 7. Update Packages

To ship an update:

1. Add an `<instructions type="update" fromversion="…">` block to `package.xml`.
2. Increment `<version>` and update `<date>`.
3. Include only the files/scripts that actually changed.
4. Re-build and upload.

The installer detects the currently installed version and executes the correct update block.

---

## 8. Uninstallation

WCS automatically reverses all PIP-managed data on uninstall (event listeners, object types, pages, language strings, …). Database schema changes are **not** automatically reversed — supply a `PackageUninstallationPlugin` or a `script` instruction that cleans up your tables if needed.

---

## 9. Useful References

* **XSD schemas** (in this repository under `XSD/`): `package.xsd`, `eventListener.xsd`, `objectType.xsd`, `page.xsd`, …
* **WoltLab Developer Docs**: <https://docs.woltlab.com/>
* **WoltLab Plugin-Store**: <https://www.woltlab.com/pluginstore/>
* [Plugin API](plugin-api.md) – event listeners, PIPs, object types
* [Module System](module-system.md) – building a full application module
