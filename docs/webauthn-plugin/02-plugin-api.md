# 02 – WCF Plugin-API – Vollständige Analyse

> Quellbasis: WoltLab Suite Core 6.2.2, Quellstand März 2026
> Relevante Dateien: `com.woltlab.wcf/package.xml`, `objectTypeDefinition.xml`, `objectType.xml`,
> `packageInstallationPlugin.xml`

---

## Inhaltsverzeichnis

- [Plugin-Paketstruktur](#plugin-paketstruktur)
- [Package Installation Plugins (PIPs)](#package-installation-plugins-pips)
- [Object Type System](#object-type-system)
- [Event-System](#event-system)
- [FormBuilder-API](#formbuilder-api)
- [Datenbank-Layer](#datenbank-layer)
- [Template-System](#template-system)
- [Flood-Control](#flood-control)
- [Session & Authentifizierung](#session--authentifizierung)

---

## Plugin-Paketstruktur

Ein WCF-Plugin ist ein `.tar`-Archiv mit einer `package.xml` an der Wurzel.

### Minimalstruktur

```
com.example.wcf.webauthn/
├── package.xml                        ← Pflicht: Paket-Metadaten + Installationsschritte
├── objectType.xml                     ← Registrierung der MFA-Methode
├── files.tar                          ← PHP-Klassen + Assets
│   └── lib/
│       └── system/user/multifactor/
│           └── WebAuthnMultifactorMethod.class.php
├── templates.tar                      ← Smarty-Templates
│   ├── multifactorManageWebauthn.tpl
│   └── multifactorAuthWebauthn.tpl
├── install.sql                        ← Datenbankschema
└── language/
    ├── de.xml
    └── en.xml
```

### package.xml (Beispiel)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<package name="com.example.wcf.webauthn"
         xmlns="http://www.woltlab.com"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://www.woltlab.com
                             http://www.woltlab.com/XSD/6.0/package.xsd">

    <packageinformation>
        <packagename>WebAuthn / Passkeys MFA</packagename>
        <packagedescription>WebAuthn/FIDO2 as a second authentication factor.</packagedescription>
        <version>1.0.0</version>
        <date>2026-01-01</date>
    </packageinformation>

    <authorinformation>
        <author>Your Name</author>
        <authorurl>https://example.com</authorurl>
    </authorinformation>

    <requiredpackages>
        <!-- WCF ≥ 5.4: MFA-Subsystem eingeführt -->
        <requiredpackage minversion="5.4.0">com.woltlab.wcf</requiredpackage>
    </requiredpackages>

    <instructions type="install">
        <instruction type="file">files.tar</instruction>
        <instruction type="template">templates.tar</instruction>
        <instruction type="objectType" />
        <instruction type="language" />
        <instruction type="sql">install.sql</instruction>
    </instructions>

    <instructions type="uninstall">
        <instruction type="sql">uninstall.sql</instruction>
    </instructions>
</package>
```

---

## Package Installation Plugins (PIPs)

PIPs sind die Installationsmechanismen für Plugin-Bestandteile.
Jede `<instruction type="...">` in der `package.xml` entspricht einem PIP.

### Relevante PIPs für ein MFA-Plugin

| PIP-Name | XML-Datei | Beschreibung |
|----------|-----------|-------------|
| `file` | – (tar-Archiv) | PHP-Klassen, Assets in `wcfsetup/install/files/` |
| `template` | – (tar-Archiv) | Smarty-Templates in `wcfsetup/install/templates/` |
| `objectType` | `objectType.xml` | Registriert neue Objekttypen (z.B. MFA-Methode) |
| `sql` | `install.sql` | Erstellt/ändert Datenbanktabellen |
| `language` | `de.xml`, `en.xml` | Sprachvariablen |
| `option` | `option.xml` | Konfigurationsoptionen (ACP-Einstellungen) |
| `userGroupOption` | `userGroupOption.xml` | Gruppenbasierte Berechtigungen |
| `eventListener` | `eventListener.xml` | Hooks in Core-Events |
| `templateListener` | `templateListener.xml` | Template-Hooks (HTML-Erweiterungen) |
| `cronjob` | `cronjob.xml` | Geplante Aufgaben |

### Vollständige PIP-Liste (WCF 6.2)

Alle verfügbaren PIPs laut `com.woltlab.wcf/packageInstallationPlugin.xml`:

```
acl, aclOption, acpMenu, acpSearchProvider, acpTemplate, bbcode, box,
clipboardAction, coreObject, cronjob, database, eventListener, file,
fileDelete, language, mediaProvider, menu, menuItem, objectType,
objectTypeDefinition, option, page, packageInstallationPlugin, script,
smiley, sql, style, template, templateDelete, templateListener,
userGroupOption, userMenu, userNotificationEvent, userOption,
userProfileMenu
```

---

## Object Type System

Das Object Type System ist das Herzstück der WCF-Erweiterbarkeit. Es erlaubt Plugins,
neue Implementierungen für definierte Schnittstellen zu registrieren.

### Konzept

```
objectTypeDefinition   →  definiert einen "Slot" + optionales Interface
        ↓
objectType             →  füllt diesen Slot mit einer konkreten Klasse
```

### objectTypeDefinition.xml (Ausschnitt – MFA-relevant)

```xml
<!-- Definiert den "MFA-Methoden-Slot" -->
<definition>
    <name>com.woltlab.wcf.multifactor</name>
    <interfacename>wcf\system\user\multifactor\IMultifactorMethod</interfacename>
</definition>

<!-- Definiert den "FloodControl-Slot" (für Rate-Limiting) -->
<definition>
    <name>com.woltlab.wcf.floodControl</name>
</definition>
```

### Vollständige Liste aller objectTypeDefinitions (WCF 6.2)

| Definition-Name | Interface |
|----------------|-----------|
| `com.woltlab.wcf.acl` | – |
| `com.woltlab.wcf.acl.simple` | – |
| `com.woltlab.wcf.attachment.objectType` | `IAttachmentObjectType` |
| `com.woltlab.wcf.boxController` | `IBoxController` |
| `com.woltlab.wcf.captcha` | `ICaptchaHandler` |
| `com.woltlab.wcf.category` | `ICategoryType` |
| `com.woltlab.wcf.clipboardItem` | – |
| `com.woltlab.wcf.comment.commentableContent` | `ICommentManager` |
| `com.woltlab.wcf.condition.box` | `IContentCondition` |
| `com.woltlab.wcf.condition.notice` | `IContentCondition` |
| `com.woltlab.wcf.condition.trophy` | `IUserCondition` |
| `com.woltlab.wcf.condition.userGroupAssignment` | `IUserCondition` |
| `com.woltlab.wcf.condition.userSearch` | `IObjectListCondition` |
| `com.woltlab.wcf.content.userContentProvider` | `IUserContentProvider` |
| `com.woltlab.wcf.deletedContent` | `IDeletedContentProvider` |
| `com.woltlab.wcf.edit.historySavingObject` | `IHistorySavingObjectTypeProvider` |
| `com.woltlab.wcf.exporter` | `IExporter` |
| `com.woltlab.wcf.file` | `IFileProcessor` |
| `com.woltlab.wcf.floodControl` | – |
| `com.woltlab.wcf.importer` | `IImporter` |
| `com.woltlab.wcf.label.object` | `ILabelObjectHandler` |
| `com.woltlab.wcf.label.objectType` | `ILabelObjectTypeHandler` |
| `com.woltlab.wcf.like.likeableObject` | `ILikeObjectTypeProvider` |
| `com.woltlab.wcf.message` | – |
| `com.woltlab.wcf.message.embeddedObject` | `IMessageEmbeddedObjectHandler` |
| `com.woltlab.wcf.message.quote` | `IMessageQuoteHandler` |
| `com.woltlab.wcf.modifiableContent` | `IExtendedModificationLogHandler` |
| `com.woltlab.wcf.moderation.activation` | `IModerationQueueActivationHandler` |
| `com.woltlab.wcf.moderation.report` | `IModerationQueueReportHandler` |
| `com.woltlab.wcf.moderation.type` | `IModerationQueueManager` |
| `com.woltlab.wcf.multifactor` | **`IMultifactorMethod`** ← relevant |
| `com.woltlab.wcf.notification.objectType` | `IUserNotificationObjectType` |
| `com.woltlab.wcf.payment.method` | `IPaymentMethod` |
| `com.woltlab.wcf.payment.type` | `IPaymentType` |
| `com.woltlab.wcf.poll` | `IPollHandler` |
| `com.woltlab.wcf.rebuildData` | `IRebuildDataWorker` |
| `com.woltlab.wcf.searchableObjectType` | – |
| `com.woltlab.wcf.sitemap.object` | `ISitemapObjectObjectType` |
| `com.woltlab.wcf.statDailyHandler` | `IStatDailyHandler` |
| `com.woltlab.wcf.tagging.taggableObject` | – |
| `com.woltlab.wcf.user.activityPointEvent` | – |
| `com.woltlab.wcf.user.objectWatch` | `IUserObjectWatch` |
| `com.woltlab.wcf.user.recentActivityEvent` | `IUserActivityEvent` |
| `com.woltlab.wcf.versionTracker.objectType` | `IVersionTrackerProvider` |
| `com.woltlab.wcf.visitTracker.objectType` | – |

### objectType.xml (Beispiel für MFA-Plugin)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<data xmlns="http://www.woltlab.com"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.woltlab.com
                          http://www.woltlab.com/XSD/6.0/objectType.xsd">
    <import>
        <!-- MFA-Methoden-Registrierung -->
        <type>
            <name>com.example.wcf.multifactor.webauthn</name>
            <definitionname>com.woltlab.wcf.multifactor</definitionname>
            <icon>key</icon>           <!-- FontAwesome Icon-Name -->
            <priority>20</priority>    <!-- Höher = bevorzugt bei mehreren Methoden -->
            <classname>wcf\system\user\multifactor\WebAuthnMultifactorMethod</classname>
        </type>

        <!-- FloodControl-Registrierung (für Rate-Limiting) -->
        <type>
            <name>com.example.wcf.multifactor.webauthn</name>
            <definitionname>com.woltlab.wcf.floodControl</definitionname>
        </type>
    </import>
</data>
```

### Priority-Werte der eingebauten MFA-Methoden (Referenz)

| Methode | Priority | Bedeutung |
|---------|---------|-----------|
| TOTP | 10 | Standard-Methode, wird zuerst angezeigt |
| E-Mail | 5 | Fallback |
| Backup-Codes | 1 | Letzter Ausweg |
| **WebAuthn (Plugin)** | **20** | Wird als erstes angezeigt (empfohlen) |

---

## Event-System

Das Event-System erlaubt Plugins, in Core-Prozesse einzuhängen, ohne Kern-Dateien zu ändern.

### eventListener.xml (Beispiel)

```xml
<listener>
    <name>com.example.wcf.webauthn.userLoggedIn</name>
    <eventclassname>wcf\event\user\authentication\UserLoggedIn</eventclassname>
    <listenerclassname>wcf\system\event\listener\WebAuthnUserLoggedInListener</listenerclassname>
</listener>
```

### Wichtige Events für MFA/Auth

| Event-Klasse | Wann gefeuert |
|-------------|--------------|
| `wcf\event\user\authentication\UserLoggedIn` | Nach erfolgreichem Login (inkl. MFA) |
| `wcf\form\MultifactorManageForm` – `saved@` | Nach erfolgreichem MFA-Setup |

### EventHandler-API

```php
use wcf\system\event\EventHandler;
use wcf\event\user\authentication\UserLoggedIn;

// In einer Listener-Klasse:
public function __invoke(UserLoggedIn $event): void
{
    $user = $event->user;
    // Eigene Logik...
}
```

---

## FormBuilder-API

WCF verwendet einen **deklarativen FormBuilder**, der Formular-Felder als Objekte modelliert.
Dies ist zentral für `IMultifactorMethod`.

### Wichtige Feld-Typen

| Klasse | Beschreibung |
|--------|-------------|
| `TextFormField` | Einzeiliges Textfeld |
| `PasswordFormField` | Passwort-Feld |
| `HiddenFormField` | Verstecktes Feld (wichtig für WebAuthn-Response!) |
| `ButtonFormField` | Schaltfläche mit Wert |
| `RadioButtonFormField` | Auswahlfeld |
| `TemplateFormNode` | Rendert ein Smarty-Template innerhalb des Formulars |
| `LanguageItemFormNode` | Zeigt einen Sprach-String an |
| `RejectEverythingFormField` | Lehnt immer ab (verhindert Formular-Submit) |

### FormContainer

```php
use wcf\system\form\builder\container\FormContainer;
use wcf\system\form\builder\field\TextFormField;
use wcf\system\form\builder\field\HiddenFormField;

$container = FormContainer::create('webauthnContainer')
    ->label('wcf.user.security.multifactor.webauthn.register')
    ->appendChildren([
        HiddenFormField::create('credentialId')
            ->objectProperty('credentialId'),
        HiddenFormField::create('clientDataJSON')
            ->objectProperty('clientDataJSON'),
        HiddenFormField::create('attestationObject')
            ->objectProperty('attestationObject'),
        TextFormField::create('deviceName')
            ->label('wcf.user.security.multifactor.webauthn.deviceName')
            ->maximumLength(200),
    ]);
$form->appendChild($container);
```

### FormFieldValidator

```php
use wcf\system\form\builder\field\validation\FormFieldValidator;
use wcf\system\form\builder\field\validation\FormFieldValidationError;

->addValidator(new FormFieldValidator('credentialValidator', function ($field) {
    $value = $field->getValue();
    if (empty($value)) {
        $field->addValidationError(new FormFieldValidationError(
            'missing',
            'wcf.user.security.multifactor.webauthn.error.missing'
        ));
    }
}))
```

### Formular-Daten lesen

```php
public function processManagementForm(IFormDocument $form, Setup $setup): mixed
{
    $formData = $form->getData();
    // $formData['data']['credentialId']
    // $formData['data']['clientDataJSON']
    // $formData['data']['attestationObject']
    // ...
}
```

---

## Datenbank-Layer

WCF verwendet ein eigenes ORM-Layer, basierend auf PDO.

### Direktes SQL (in MFA-Methoden üblich)

```php
use wcf\system\WCF;

// SELECT
$sql = "SELECT  *
        FROM    wcf1_user_multifactor_webauthn
        WHERE   setupID = ?
            AND credentialId = ?";
$statement = WCF::getDB()->prepare($sql);
$statement->execute([$setupId, $credentialId]);
$row = $statement->fetchSingleRow();

// INSERT
$sql = "INSERT INTO wcf1_user_multifactor_webauthn
                    (setupID, credentialId, publicKey, signCount, deviceName, createTime)
        VALUES      (?, ?, ?, ?, ?, ?)";
$statement = WCF::getDB()->prepare($sql);
$statement->execute([$setupId, $credentialId, $publicKey, 0, $deviceName, TIME_NOW]);

// UPDATE mit FOR UPDATE (row-level lock für Replay-Schutz)
$sql = "UPDATE  wcf1_user_multifactor_webauthn
        SET     signCount = ?
        WHERE   setupID = ?
            AND credentialId = ?
            AND signCount < ?";
$statement = WCF::getDB()->prepare($sql);
$statement->execute([$newCount, $setupId, $credentialId, $newCount]);
if ($statement->getAffectedRows() !== 1) {
    throw new \RuntimeException('Replay-Angriff erkannt oder Credential nicht gefunden.');
}
```

### Transaktionen

```php
WCF::getDB()->beginTransaction();
try {
    // ... Datenbankoperationen ...
    WCF::getDB()->commitTransaction();
} catch (\Exception $e) {
    WCF::getDB()->rollBackTransaction();
    throw $e;
}
```

> **Hinweis:** `processManagementForm()` und `processAuthenticationForm()` werden **innerhalb
> einer Transaktion** aufgerufen (durch `MultifactorManageForm` / `MultifactorAuthenticationForm`).
> Kein eigenes `beginTransaction()` nötig.

### Tabellen-Prefix

WCF verwendet den Prefix `wcf1_` (konfigurierbar bei Installation). Eigene Plugin-Tabellen
sollen ebenfalls diesen Prefix verwenden – in SQL-Dateien wird `wcf1_` direkt geschrieben;
WCF ersetzt diesen automatisch durch den konfigurierten Prefix.

### DatabaseObject (ORM-Basisklasse)

Für einfachere CRUD-Operationen kann `DatabaseObject` genutzt werden:

```php
use wcf\data\DatabaseObject;

class WebAuthnCredential extends DatabaseObject
{
    protected static $databaseTableName = 'user_multifactor_webauthn';
    protected static $databaseTableIndexName = 'credentialID';
}
```

---

## Template-System

WCF verwendet **Smarty** als Template-Engine.

### Templates für MFA-Methoden

Templates liegen in `templates/` und werden mit dem PIP `template` installiert.
Namenskonvention: `multifactorManage{MethodName}.tpl` und `multifactorAuth{MethodName}.tpl`

### Template in FormBuilder einbinden

```php
use wcf\system\form\builder\TemplateFormNode;

$form->appendChild(
    TemplateFormNode::create('webauthnUI')
        ->templateName('multifactorManageWebauthn')
        ->variables([
            'credentials' => $credentials,
            'challenge'   => $challenge,
        ])
);
```

### Beispiel-Template (Smarty)

```smarty
{* templates/multifactorManageWebauthn.tpl *}
<script>
    require(['WoltLabSuite/Core/Mfa/WebAuthn'], function(WebAuthn) {
        WebAuthn.init({
            challenge: '{$challenge|encodeJS}',
            rpId: '{$__wcf->getPath()|encodeJS}',
            userId: {$__wcf->getUser()->userID}
        });
    });
</script>

<div class="webauthnContainer">
    <p>{lang}wcf.user.security.multifactor.webauthn.explanation{/lang}</p>
    <button type="button" id="webauthnRegister" class="button">
        {lang}wcf.user.security.multifactor.webauthn.registerDevice{/lang}
    </button>
</div>
```

---

## Flood-Control

WCF bietet ein eingebautes Rate-Limiting über `FloodControl`.

### Registrierung (objectType.xml)

```xml
<type>
    <name>com.example.wcf.multifactor.webauthn</name>
    <definitionname>com.woltlab.wcf.floodControl</definitionname>
</type>
```

### Nutzung in PHP

```php
use wcf\system\flood\FloodControl;

// Fehlversuch registrieren
FloodControl::getInstance()->registerUserContent(
    'com.example.wcf.multifactor.webauthn',
    $setup->getId()
);

// Versuche zählen
$attempts = FloodControl::getInstance()->countUserContent(
    'com.example.wcf.multifactor.webauthn',
    $setup->getId(),
    new \DateInterval('PT10M')  // Letzte 10 Minuten
);

if ($attempts['count'] > 5) {
    $field->addValidationError(new FormFieldValidationError(
        'flood',
        'wcf.user.security.multifactor.webauthn.error.flood',
        $attempts
    ));
    return;
}
```

---

## Session & Authentifizierung

### MFA-Login-Flow (SessionHandler)

```php
// In LoginForm::save():
$needsMultifactor = WCF::getSession()->changeUserAfterMultifactorAuthentication($user);

if ($needsMultifactor) {
    // User hat MFA aktiv → noch NICHT eingeloggt
    // Session speichert: pendingUserId + expires
    // Redirect → MultifactorAuthenticationForm
} else {
    // User hat kein MFA → direkt eingeloggt
    WCF::getSession()->registerReauthentication();
    EventHandler::getInstance()->fire(new UserLoggedIn($user));
}
```

### Nach erfolgreicher MFA (MultifactorAuthenticationForm::save())

```php
WCF::getSession()->applyPendingUserChange($this->user);
// → ruft intern changeUser($user) auf
// → User ist jetzt vollständig eingeloggt

EventHandler::getInstance()->fire(new UserLoggedIn($this->user));
```

### Reauthentifizierung (für Management-Forms)

Das MFA-Management erfordert eine **Re-Authentifizierung** (Passwort eingeben), auch wenn
der User bereits eingeloggt ist. Dies wird über `TReauthenticationCheck` realisiert:

```php
use wcf\system\user\authentication\TReauthenticationCheck;

class MultifactorManageForm extends AbstractFormBuilderForm
{
    use TReauthenticationCheck;

    public function checkPermissions()
    {
        parent::checkPermissions();
        $this->requestReauthentication(
            LinkHandler::getInstance()->getControllerLink(static::class, [
                'object' => $this->method,
            ])
        );
    }
}
```

> Dieser Mechanismus ist bereits in `MultifactorManageForm` eingebaut – das WebAuthn-Plugin
> muss ihn **nicht selbst** implementieren, da es `MultifactorManageForm` als Basis nutzt.
