# 07 – WCF Plugin-API – Allgemeine Einführung & Plugin-Ideen

> Diese Seite gibt eine **technologisch neutrale Übersicht** der WoltLab Suite Core Plugin-API –
> losgelöst vom WebAuthn-Kontext. Sie erklärt, was mit Plugins möglich ist, welche
> Erweiterungspunkte existieren und enthält einen **Katalog konkreter Plugin-Ideen**.
>
> WCF-Version: 6.2.x · Quellstand März 2026

---

## Inhaltsverzeichnis

- [Das Plugin-System im Überblick](#das-plugin-system-im-überblick)
- [Die drei Säulen der WCF-Erweiterbarkeit](#die-drei-säulen-der-wcf-erweiterbarkeit)
  - [1. Object Type System (Implementierungen)](#1-object-type-system-implementierungen)
  - [2. Event-System (Hooks)](#2-event-system-hooks)
  - [3. Template-Listener (HTML-Erweiterungen)](#3-template-listener-html-erweiterungen)
- [Weitere Bausteine](#weitere-bausteine)
  - [Cronjobs](#cronjobs)
  - [Background Jobs](#background-jobs)
  - [ACP-Menü & Optionen](#acp-menü--optionen)
  - [Sprachvariablen & Übersetzungen](#sprachvariablen--übersetzungen)
  - [Datenbank-PIPs](#datenbank-pips)
  - [Seiten (Pages)](#seiten-pages)
- [Zusammenspiel der Mechanismen](#zusammenspiel-der-mechanismen)
- [Plugin-Ideen-Katalog](#plugin-ideen-katalog)
  - [Sicherheit & Authentifizierung](#sicherheit--authentifizierung)
  - [Spam-Schutz & Bots](#spam-schutz--bots)
  - [Integrations-Plugins](#integrations-plugins)
  - [Benutzer & Community](#benutzer--community)
  - [Inhalte & Medien](#inhalte--medien)
  - [Administration & Monitoring](#administration--monitoring)
- [Entscheidungsbaum: Welcher Mechanismus passt?](#entscheidungsbaum-welcher-mechanismus-passt)

---

## Das Plugin-System im Überblick

WCF ist nach dem **Open/Closed-Prinzip** gebaut: Der Core ist geschlossen für Änderungen,
aber offen für Erweiterungen. Ein Plugin ändert **niemals Core-Dateien** – es registriert
Erweiterungen über ein standardisiertes Installationssystem.

```
Plugin (.tar-Archiv)
│
├── package.xml          ← Metadaten + Installationsschritte
├── *.xml                ← Konfigurationsdateien (objectType, eventListener, ...)
├── files.tar            ← PHP-Klassen, Assets
├── templates.tar        ← Smarty-Templates
└── language/            ← Sprachdateien (de.xml, en.xml, ...)
```

**Vorteil dieses Ansatzes:**
- Plugins überleben Core-Updates (keine Merge-Konflikte)
- Mehrere Plugins können dieselben Hooks nutzen, ohne sich zu stören
- Deinstallation ist sauber und vollständig (kein Code-Residuum)

---

## Die drei Säulen der WCF-Erweiterbarkeit

### 1. Object Type System (Implementierungen)

**Wann nutzen:** Wenn WCF einen definierten „Steckplatz" (Definition) bereitstellt, der mit
einer eigenen Klasse gefüllt werden soll.

```
objectTypeDefinition  (Core)     objectType  (Plugin)
┌────────────────────────┐       ┌──────────────────────────┐
│ Name: com.woltlab.wcf. │◄──────│ Name: com.mein.plugin.x  │
│        multifactor     │       │ Interface: implementiert  │
│ Interface: IMfaMethod  │       │ Klasse: MeineMfaMethode   │
└────────────────────────┘       └──────────────────────────┘
```

**Registrierung** in `objectType.xml`:
```xml
<type>
    <name>com.mein.plugin.meinemethode</name>
    <definitionname>com.woltlab.wcf.multifactor</definitionname>
    <classname>wcf\system\user\multifactor\MeineMfaMethode</classname>
    <priority>20</priority>
</type>
```

**Alle nutzbaren Definitions-Steckplätze** (Auswahl der wichtigsten):

| Definition | Interface | Verwendung |
|-----------|-----------|-----------|
| `com.woltlab.wcf.multifactor` | `IMultifactorMethod` | Neue MFA-Methode (WebAuthn, Hardware-OTP, …) |
| `com.woltlab.wcf.captcha` | `ICaptchaHandler` | Neues CAPTCHA (Turnstile, hCaptcha, …) |
| `com.woltlab.wcf.notification.objectType` | `IUserNotificationObjectType` | Neue Benachrichtigungs-Quelle |
| `com.woltlab.wcf.searchableObjectType` | `ISearchableObjectType` | Neue suchbare Inhalte |
| `com.woltlab.wcf.comment.commentableContent` | `ICommentManager` | Kommentare für eigene Objekte |
| `com.woltlab.wcf.boxController` | `IBoxController` | Neue Widget-Box im Frontend |
| `com.woltlab.wcf.sitemap.object` | `ISitemapObjectObjectType` | Objekte in Sitemap.xml |
| `com.woltlab.wcf.rebuildData` | `IRebuildDataWorker` | Neuaufbau-Worker für eigene Daten |
| `com.woltlab.wcf.statDailyHandler` | `IStatDailyHandler` | Statistiken im ACP-Dashboard |
| `com.woltlab.wcf.floodControl` | – | Rate-Limiting für eigene Aktionen |
| `com.woltlab.wcf.importer` | `IImporter` | Import aus anderen Systemen |
| `com.woltlab.wcf.exporter` | `IExporter` | Datenexport |
| `com.woltlab.wcf.payment.method` | `IPaymentMethod` | Neue Zahlungsmethode |
| `com.woltlab.wcf.clipboardItem` | – | Clipboard-Aktionen für eigene Listen |
| `com.woltlab.wcf.condition.*` | `IContentCondition` / `IUserCondition` | Bedingungen für Boxen, Trophäen, Gruppen |

---

### 2. Event-System (Hooks)

**Wann nutzen:** Wenn man in einen laufenden Core-Prozess einhaken will, ohne dessen
Klasse zu verändern. Events werden mit `EventHandler::getInstance()->fire(new MeinEvent())`
ausgelöst und können in beliebig vielen Listenern empfangen werden.

**Registrierung** in `eventListener.xml`:
```xml
<listener>
    <name>com.mein.plugin.meinlistener</name>
    <eventclassname>wcf\event\user\authentication\UserLoggedIn</eventclassname>
    <listenerclassname>wcf\system\event\listener\MeinLoginListener</listenerclassname>
</listener>
```

**Listener-Klasse:**
```php
namespace wcf\system\event\listener;

use wcf\event\user\authentication\UserLoggedIn;

final class MeinLoginListener
{
    public function __invoke(UserLoggedIn $event): void
    {
        $user = $event->getUser();
        // Eigene Logik nach erfolgreichem Login
    }
}
```

#### Vollständige Liste der verfügbaren Events (WCF 6.2, 145 Event-Klassen)

**Authentifizierung & Benutzer**

| Event-Klasse | Auslöser |
|-------------|---------|
| `wcf\event\user\authentication\UserLoggedIn` | Nach jedem erfolgreichen Login (inkl. MFA) |
| `wcf\event\user\authentication\configuration\ConfigurationLoading` | Auth-Konfiguration wird geladen |
| `wcf\event\user\RegistrationSpamChecking` | Registrierung wird auf Spam geprüft |
| `wcf\event\user\UsernameValidating` | Benutzername wird validiert |
| `wcf\event\user\AvatarChanged` | Avatar wurde geändert |
| `wcf\event\user\CoverPhotoChanged` | Titelbild wurde geändert |
| `wcf\event\user\UserFollowed` | Nutzer folgt einem anderen |
| `wcf\event\user\UserUnfollowed` | Nutzer entfolgt |

**Kommentare**

| Event-Klasse | Auslöser |
|-------------|---------|
| `wcf\event\comment\CommentCreated` | Kommentar erstellt |
| `wcf\event\comment\CommentPublished` | Kommentar freigeschaltet |
| `wcf\event\comment\CommentUpdated` | Kommentar bearbeitet |
| `wcf\event\comment\CommentsDeleted` | Kommentar(e) gelöscht |
| `wcf\event\comment\response\ResponseCreated` | Kommentar-Antwort erstellt |
| `wcf\event\comment\response\ResponsePublished` | Antwort freigeschaltet |

**Nachrichten & Spam**

| Event-Klasse | Auslöser |
|-------------|---------|
| `wcf\event\message\MessageSpamChecking` | Nachricht wird auf Spam geprüft |
| `wcf\event\page\ContactFormSpamChecking` | Kontaktformular wird geprüft |

**Dateien & Medien**

| Event-Klasse | Auslöser |
|-------------|---------|
| `wcf\event\file\UploadCompleted` | Datei-Upload abgeschlossen |
| `wcf\event\file\DownloadStarting` | Download beginnt |
| `wcf\event\file\GenerateThumbnail` | Thumbnail wird generiert |
| `wcf\event\file\GenerateWebpVariant` | WebP-Variante wird generiert |

**Sprache & Cache**

| Event-Klasse | Auslöser |
|-------------|---------|
| `wcf\event\language\PhraseChanged` | Sprachphrase geändert |
| `wcf\event\language\LanguageImported` | Sprachpaket importiert |
| `wcf\event\language\PreloadPhrasesCollecting` | Preload-Phrasen werden gesammelt |
| `wcf\event\cache\CacheCleared` | Cache wurde geleert |

**Paket & Plugins**

| Event-Klasse | Auslöser |
|-------------|---------|
| `wcf\event\package\PackageListChanged` | Paketliste geändert |
| `wcf\event\package\PackageInstallationPluginSynced` | PIP synchronisiert |
| `wcf\event\package\PackageUninstallationStarted` | Deinstallation beginnt |

**Moderation**

| Event-Klasse | Auslöser |
|-------------|---------|
| `wcf\event\moderation\DeletedContentProvidersCollecting` | Gelöschte Inhalte werden gesammelt |
| `wcf\event\moderation\queue\UserAssigned` | Moderationsaufgabe zugewiesen |

**Diverse**

| Event-Klasse | Auslöser |
|-------------|---------|
| `wcf\event\session\PreserveVariablesCollecting` | Session-Variablen werden beibehalten |
| `wcf\event\request\ActivePageResolving` | Aktive Seite wird bestimmt |
| `wcf\event\spider\SpiderCollecting` | Bot-Liste wird aufgebaut |
| `wcf\event\style\StyleCopied` | Style wurde kopiert |
| `wcf\event\worker\RebuildWorkerCollecting` | Rebuild-Worker werden gesammelt |

> **Hinweis:** Es gibt zusätzlich ältere Events über `EventHandler::fireAction($object, $event)`
> (Legacy-System). In WCF 6.x wird das neue PSR-14-basierte System (`EventHandler::fire()`) bevorzugt.

---

### 3. Template-Listener (HTML-Erweiterungen)

**Wann nutzen:** Wenn man HTML in ein bestehendes Template einfügen will, ohne es zu ersetzen.

```xml
<!-- In templateListener.xml -->
<templatelistener name="com.mein.plugin.userProfileExtension">
    <environment>user</environment>
    <templatename>userProfile</templatename>
    <eventname>afterMembership</eventname>
    <templatecode><![CDATA[
        {if $userProfile->userID}
            <section class="section">
                <h2 class="sectionTitle">Meine Erweiterung</h2>
                <p>{$extraData}</p>
            </section>
        {/if}
    ]]></templatecode>
</templatelistener>
```

**Parameter:**
- `environment`: `user` (Frontend) oder `admin` (ACP)
- `templatename`: Name des Templates (ohne `.tpl`)
- `eventname`: Name des Event-Hooks im Template (`{event name='eventname'}`)
- `templatecode`: Inline-Smarty-Code **oder** `{include file='meinTemplate'}`

---

## Weitere Bausteine

### Cronjobs

Wiederkehrende Aufgaben, die nach einem Zeitplan ausgeführt werden.

**Implementierung:**
```php
namespace wcf\system\cronjob;

use wcf\data\cronjob\Cronjob;

final class MeinCronjob implements ICronjob
{
    public function execute(Cronjob $cronjob): void
    {
        // z.B. abgelaufene Einträge aufräumen
        $sql = "DELETE FROM wcf1_mein_plugin_daten WHERE expires < ?";
        $statement = WCF::getDB()->prepare($sql);
        $statement->execute([TIME_NOW]);
    }
}
```

**Registrierung** in `cronjob.xml`:
```xml
<cronjob name="com.mein.plugin.cleanup">
    <classname>wcf\system\cronjob\MeinCronjob</classname>
    <startMinute>0</startMinute>
    <startHour>3</startHour>    <!-- Täglich um 3 Uhr -->
    <startDom>*</startDom>
    <startMonth>*</startMonth>
    <startDow>*</startDow>
    <canBeDisabled>1</canBeDisabled>
    <canBeEdited>1</canBeEdited>
</cronjob>
```

### Background Jobs

Für zeitintensive Aufgaben, die asynchron im Hintergrund laufen (z.B. E-Mail-Versand,
aufwändige Berechnungen).

```php
namespace wcf\system\background\job;

final class MeinHintergrundJob extends AbstractBackgroundJob
{
    public function __construct(private readonly int $userId) {}

    public function perform(): void
    {
        // Asynchrone Logik
    }
}

// Ausführung planen:
BackgroundQueueHandler::getInstance()->enqueueIn(
    new MeinHintergrundJob($userId),
    60 // In 60 Sekunden ausführen
);

// Sofort ausführen:
BackgroundQueueHandler::getInstance()->performJob(
    new MeinHintergrundJob($userId)
);
```

### ACP-Menü & Optionen

**Neuen Menüpunkt im ACP hinzufügen** (`acpMenu.xml` im Plugin, oder via PIP):
```xml
<!-- In der Plugin-package.xml mit PIP acpMenu: -->
```

In der Praxis werden ACP-Menüpunkte über das `page`-PIP und `acpMenu`-PIP kombiniert.
Ein neues ACP-Modul besteht typisch aus:
1. `page.xml` → Registriert die Controller-URL
2. `acpMenu.xml` → Fügt den Menüeintrag ein
3. PHP-Klasse in `files.tar` → Zeigt das ACP-Formular/die Liste

**Konfigurationsoptionen** (`option.xml`):
```xml
<option name="mein_plugin_max_devices">
    <categoryname>security</categoryname>
    <optiontype>integer</optiontype>
    <defaultvalue>5</defaultvalue>
</option>
```
Zugriff im Code: `MAX_DEVICES` (konstante aus option-Name in Großbuchstaben)

### Sprachvariablen & Übersetzungen

WCF unterstützt beliebig viele Sprachen. Sprachvariablen werden in `language/de.xml` usw. definiert.

```xml
<language languagecode="de">
    <category name="wcf.mein.plugin">
        <item name="wcf.mein.plugin.beispiel">
            <![CDATA[Hallo {$name}!]]>
        </item>
        <item name="wcf.mein.plugin.plural">
            <![CDATA[{#$count} {if $count == 1}Element{else}Elemente{/if}]]>
        </item>
    </category>
</language>
```

Zugriff in PHP:
```php
WCF::getLanguage()->getDynamicVariable('wcf.mein.plugin.beispiel', ['name' => 'Welt']);
```

Zugriff in Templates:
```smarty
{lang name='Welt'}wcf.mein.plugin.beispiel{/lang}
```

### Datenbank-PIPs

| PIP | Beschreibung |
|-----|-------------|
| `sql` | Führt SQL-Datei aus (für CREATE TABLE etc.) |
| `database` | Führt PHP-Datenbankskript aus (für komplexe Migrationen) |

**`install.sql`-Datei** wird mit `<instruction type="sql">install.sql</instruction>` ausgeführt.

```sql
CREATE TABLE IF NOT EXISTS wcf1_mein_plugin_daten (
    id          INT(10) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    userID      INT(10) NOT NULL,
    daten       TEXT    NOT NULL,
    createTime  INT(10) NOT NULL,
    expires     INT(10) DEFAULT NULL
);
ALTER TABLE wcf1_mein_plugin_daten
    ADD FOREIGN KEY (userID) REFERENCES wcf1_user (userID) ON DELETE CASCADE;
```

### Seiten (Pages)

Neue Frontend- oder ACP-Seiten können über das `page`-PIP registriert werden.
Eine Seite = eine PHP-Klasse, die `AbstractPage` oder `AbstractForm` erweitert.

```php
namespace wcf\page;

use wcf\page\AbstractAuthedPage;
use wcf\system\WCF;

final class MeineSeite extends AbstractAuthedPage
{
    public string $templateName = 'meinTemplate';

    public function assignVariables(): void
    {
        parent::assignVariables();
        WCF::getTPL()->assign(['daten' => $this->ladeDaten()]);
    }
}
```

---

## Zusammenspiel der Mechanismen

Ein reales Plugin kombiniert mehrere Mechanismen:

```
Beispiel: "Login-Protokoll" Plugin

objectType (floodControl)  → Rate-Limiting für fehlgeschlagene Logins
eventListener              → auf UserLoggedIn + LoginFailed reagieren
sql                        → Tabelle wcf1_login_log erstellen
cronjob                    → Alte Einträge täglich aufräumen
page (ACP)                 → ACP-Seite zum Anzeigen der Logs
acpMenu                    → Menüeintrag "Login-Protokoll"
language                   → Übersetzungen
userGroupOption            → Berechtigung "Kann Login-Log sehen"
```

---

## Plugin-Ideen-Katalog

### Sicherheit & Authentifizierung

#### 🔑 WebAuthn / FIDO2 MFA (Hauptfokus dieser Doku)
- **Mechanismus:** `objectType` (com.woltlab.wcf.multifactor)
- **Interface:** `IMultifactorMethod`
- **Aufwand:** Mittel–Hoch (WebAuthn-Bibliothek + TypeScript-Frontend)
- **Nutzen:** Passwortlose Security Keys / Passkeys als zweiter Faktor

#### 📱 TOTP-Migration / Backup-Tools
- **Mechanismus:** `eventListener` auf `MultifactorManageForm`-Events
- **Aufwand:** Gering
- **Nutzen:** Export/Import von TOTP-Geräten zwischen Systemen

#### 🔒 Login-Protokoll
- **Mechanismus:** `eventListener` (UserLoggedIn), `sql`, `page`, `acpMenu`, `cronjob`
- **Interface:** –
- **Aufwand:** Gering
- **Nutzen:** Alle Logins (IP, Uhrzeit, Browser, Methode) protokollieren

#### 🛡️ IP-Blockliste / Geoblocking
- **Mechanismus:** `eventListener` (Request), `sql`, `option`, `acpMenu`
- **Aufwand:** Mittel
- **Nutzen:** Bestimmte IPs oder Länder blockieren

#### 🔐 Passwortrichtlinien-Plugin
- **Mechanismus:** `eventListener` (Registrierung/Passwortänderung), `option`
- **Aufwand:** Gering
- **Nutzen:** Eigene Passwortregeln (Länge, Zeichenklassen, HaveIBeenPwned-Check)

#### 🕐 Inaktivitäts-Ausloggen
- **Mechanismus:** `eventListener` (Session), `option`
- **Aufwand:** Gering
- **Nutzen:** Automatisches Ausloggen nach X Minuten Inaktivität

---

### Spam-Schutz & Bots

#### 🚫 Honeypot-CAPTCHA
- **Mechanismus:** `objectType` (com.woltlab.wcf.captcha)
- **Interface:** `ICaptchaHandler`
- **Aufwand:** Gering
- **Nutzen:** Unsichtbares Honeypot-Feld als Alternative zu visuellen CAPTCHAs

#### ☁️ Cloudflare Turnstile Integration
- **Mechanismus:** `objectType` (com.woltlab.wcf.captcha)
- **Interface:** `ICaptchaHandler`
- **Aufwand:** Gering–Mittel
- **Nutzen:** Datenschutzfreundliches CAPTCHA von Cloudflare

#### 📧 Wegwerf-E-Mail-Blocker
- **Mechanismus:** `eventListener` (RegistrationSpamChecking)
- **Aufwand:** Gering (API-Abfrage zu einem Dienst wie mailcheck.ai)
- **Nutzen:** Blockiert Registrierungen mit temporären E-Mail-Adressen

#### 🤖 KI-basierter Spam-Filter
- **Mechanismus:** `eventListener` (MessageSpamChecking, RegistrationSpamChecking)
- **Aufwand:** Mittel
- **Nutzen:** KI-API (OpenAI, lokales Modell) analysiert Nachrichten auf Spam

---

### Integrations-Plugins

#### 🔗 OAuth2 / Single Sign-On
- **Mechanismus:** `eventListener` (Auth), eigene Login-Seite, `sql`
- **Aufwand:** Hoch
- **Nutzen:** Login via GitHub, GitLab, Keycloak, LDAP etc.

> **Hinweis:** WCF bringt von Haus aus keine SSO-Integration mit. Das Plugin würde
> einen eigenen Login-Controller bereitstellen, der nach erfolgreicher OAuth2-Authentifizierung
> `SessionHandler::changeUserAfterMultifactorAuthentication()` aufruft.

#### 📊 Analytics-Integration
- **Mechanismus:** `templateListener` (head/bodyEnd-Event)
- **Aufwand:** Sehr gering
- **Nutzen:** Matomo, Plausible, Umami oder GA4 in alle Seiten einbinden

#### 🔔 Push-Benachrichtigungen (WebPush)
- **Mechanismus:** `eventListener` (UserNotification), `sql`, `page`, `cronjob`
- **Aufwand:** Hoch
- **Nutzen:** Browser-Push-Notifications für neue Benachrichtigungen

#### 🗃️ S3-Dateiablage
- **Mechanismus:** `eventListener` (file/UploadCompleted), `option`
- **Aufwand:** Mittel
- **Nutzen:** Hochgeladene Dateien automatisch zu AWS S3 / MinIO / Backblaze spiegeln

#### 📨 Discord-Webhook-Integration
- **Mechanismus:** `eventListener` (verschiedene Events), `option`
- **Aufwand:** Gering
- **Nutzen:** Neue Beiträge/Kommentare/Registrierungen an Discord posten

---

### Benutzer & Community

#### 🏅 Erweiterte Profilfelder
- **Mechanismus:** `userOption`, `templateListener` (userProfile)
- **Aufwand:** Gering
- **Nutzen:** Zusätzliche Felder im Benutzerprofil (Social Links, Pronomen, etc.)

#### 📊 Benutzer-Statistiken
- **Mechanismus:** `objectType` (com.woltlab.wcf.statDailyHandler), `sql`
- **Interface:** `IStatDailyHandler`
- **Aufwand:** Gering–Mittel
- **Nutzen:** Eigene Datenpunkte ins ACP-Statistik-Dashboard einbauen

#### 🎯 Benutzersegmentierung / Tags
- **Mechanismus:** `eventListener`, `sql`, `acpMenu`, `userGroupOption`
- **Aufwand:** Mittel
- **Nutzen:** Benutzer intern taggen (z.B. "Beta-Tester", "VIP")

#### 📬 Willkommens-E-Mail Customizer
- **Mechanismus:** `eventListener` (UserLoggedIn, Registrierung), `option`
- **Aufwand:** Gering
- **Nutzen:** Eigene Willkommens-E-Mails mit personalisierten Inhalten

---

### Inhalte & Medien

#### 🔍 Erweiterte Volltextsuche
- **Mechanismus:** `objectType` (com.woltlab.wcf.searchableObjectType)
- **Interface:** `ISearchableObjectType` / `ISearchProvider`
- **Aufwand:** Mittel
- **Nutzen:** Externe Suchmotoren anbinden (Elasticsearch, Meilisearch)

#### 📄 PDF-Export
- **Mechanismus:** `eventListener` (verschiedene Objekte), eigene Seite
- **Aufwand:** Mittel
- **Nutzen:** Artikel, Beiträge als PDF exportieren

#### 🗺️ Sitemap-Erweiterung
- **Mechanismus:** `objectType` (com.woltlab.wcf.sitemap.object)
- **Interface:** `ISitemapObjectObjectType`
- **Aufwand:** Gering
- **Nutzen:** Eigene Inhalte in die automatisch generierte sitemap.xml aufnehmen

#### 📦 Medien-CDN-Proxy
- **Mechanismus:** `eventListener` (file/DownloadStarting), `option`
- **Aufwand:** Mittel
- **Nutzen:** Dateidownloads über ein CDN ausliefern

---

### Administration & Monitoring

#### 📈 ACP-Dashboard-Widgets
- **Mechanismus:** `eventListener` (BoxCollecting), `objectType` (com.woltlab.wcf.boxController)
- **Interface:** `IBoxController`
- **Aufwand:** Gering
- **Nutzen:** Eigene Informations-Widgets auf dem ACP-Dashboard

#### 🔧 Rebuild-Worker
- **Mechanismus:** `objectType` (com.woltlab.wcf.rebuildData)
- **Interface:** `IRebuildDataWorker`
- **Aufwand:** Gering–Mittel
- **Nutzen:** Eigene Daten können über ACP > Wartung neu aufgebaut werden

#### 📋 Audit-Log
- **Mechanismus:** `eventListener` (diverse), `sql`, `acpMenu`, `cronjob`, `page`
- **Aufwand:** Mittel
- **Nutzen:** Alle sensitiven Admin-Aktionen protokollieren

#### ⚠️ Integritäts-Checker
- **Mechanismus:** `cronjob`, `eventListener` (PackageListChanged)
- **Aufwand:** Mittel
- **Nutzen:** Regelmäßige Prüfung von Datenbankintegrität, Dateisystem-Checksums

---

## Entscheidungsbaum: Welcher Mechanismus passt?

```
Ich möchte ...

... eine neue Implementierung für einen WCF-Slot bereitstellen
    (neue MFA-Methode, neues CAPTCHA, neue Suche, neue Box, ...)
    → objectType + Interface implementieren

... auf ein bestehendes WCF-Ereignis reagieren
    (Login, Upload, Kommentar, Spam-Prüfung, ...)
    → eventListener + PHP-Listener-Klasse

... HTML in ein bestehendes Template einfügen
    (Profil-Erweiterung, neues Formularfeld, Footer-Script, ...)
    → templateListener

... eine regelmäßige Aufgabe ausführen
    (Cleanup, E-Mail-Versand, Sync, ...)
    → cronjob

... eine zeitintensive Aufgabe asynchron ausführen
    (API-Aufruf, Konvertierung, Versand, ...)
    → AbstractBackgroundJob

... eine neue Seite / ein neues ACP-Modul hinzufügen
    → page PIP + AbstractPage/AbstractForm + acpMenu

... globale Einstellungen für das Plugin bereitstellen
    → option PIP (ACP-Einstellungen)

... benutzergruppenbasierte Berechtigungen hinzufügen
    → userGroupOption PIP

... Datenbanktabellen anlegen
    → sql PIP (install.sql + uninstall.sql)

... Texte/Übersetzungen hinzufügen
    → language PIP (de.xml, en.xml, ...)
```

---

## Weiterführende Links in dieser Dokumentation

- [02 – Plugin-API im Detail](02-plugin-api.md) – Vollständige Referenz aller PIPs, FormBuilder, DB-Layer
- [03 – MFA-Subsystem](03-mfa-subsystem.md) – Tiefenanalyse für MFA-spezifische Plugins
- [04 – WebAuthn-Implementierung](04-webauthn-implementierung.md) – Konkretes Beispiel-Plugin
