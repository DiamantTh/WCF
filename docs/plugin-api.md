# Plugin API

WoltLab Suite Core offers several complementary mechanisms for extending its functionality without touching the core source code.

---

## 1. Event System

### Overview

WCS ships a modern **PSR-14-compatible event dispatcher** (`wcf\system\event\EventHandler`). Every important action in the core fires an event that your listener can react to.

### Registering a listener via `eventListener.xml`

Create the file `eventListener.xml` inside your package:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<data xmlns="http://www.woltlab.com"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.woltlab.com http://www.woltlab.com/XSD/6.0/eventListener.xsd">
    <import>
        <eventlistener>
            <!-- The fully-qualified class name of the PSR-14 event object -->
            <eventclassname>wcf\system\user\authentication\event\UserLoggedIn</eventclassname>
            <!-- Your listener class -->
            <listenerclassname>com\example\mypkg\system\event\listener\MyLoginListener</listenerclassname>
            <!-- "user" | "admin" | "all" -->
            <environment>all</environment>
        </eventlistener>
    </import>
</data>
```

Reference this file from your `package.xml`:

```xml
<instruction type="eventListener" />
```

### Writing a PSR-14 listener

```php
<?php

namespace com\example\mypkg\system\event\listener;

use wcf\system\user\authentication\event\UserLoggedIn;

final class MyLoginListener
{
    public function __invoke(UserLoggedIn $event): void
    {
        $user = $event->user;
        // your logic here
    }
}
```

### Legacy event listeners (pre-5.5)

For events that still use the legacy `IParameterizedEventListener` interface, implement the `execute()` method:

```php
<?php

namespace com\example\mypkg\system\event\listener;

use wcf\system\event\listener\IParameterizedEventListener;

final class MyLegacyListener implements IParameterizedEventListener
{
    public function execute($eventObj, $className, $eventName, array &$parameters): void
    {
        // react to the event
    }
}
```

---

## 2. Package Installation Plugins (PIPs)

A PIP is a handler that is called during package installation, update, or uninstallation for a specific instruction type. WCS ships with built-in PIPs for common tasks:

| PIP instruction type | Purpose |
|---|---|
| `file` | Install/overwrite arbitrary files under the WCS root |
| `template` | Install Smarty templates |
| `acpTemplate` | Install ACP Smarty templates |
| `eventListener` | Register event listeners |
| `objectType` | Register object types |
| `objectTypeDefinition` | Register object-type definitions |
| `database` | Run database migrations via PHP scripts |
| `language` | Install language strings |
| `option` | Add global configuration options |
| `userGroupOption` | Add per-group permissions |
| `cronjob` | Register background cron jobs |
| `page` | Register front-end pages |
| `menu` / `menuItem` | Add navigation entries |
| `box` | Register content boxes |
| `acpMenu` | Add ACP navigation entries |
| `style` | Install styles |
| `bbcode` | Register BBCode elements |
| `clipboardAction` | Add clipboard (multi-select) actions |
| `script` | Execute arbitrary PHP scripts during install/update |

### Writing a custom PIP

1. Create a class that extends `AbstractXMLPackageInstallationPlugin` (or `AbstractPackageInstallationPlugin` for non-XML data).
2. Register it via the built-in `packageInstallationPlugin` PIP in your `package.xml`.

```xml
<instruction type="packageInstallationPlugin" />
```

Backed by the file `packageInstallationPlugin.xml`:

```xml
<data xmlns="http://www.woltlab.com" …>
    <import>
        <packageinstallationplugin>
            <pluginname>myCustomPIP</pluginname>
            <classname>com\example\mypkg\system\package\plugin\MyCustomPIPPackageInstallationPlugin</classname>
        </packageinstallationplugin>
    </import>
</data>
```

---

## 3. Object Types and Definitions

The **object-type system** is the primary extension point for attaching new behaviour to existing domain objects (users, content items, attachments, …).

### Object-Type Definition

An *object-type definition* declares an extension point (e.g. `com.woltlab.wcf.user.activityPoint`). Usually defined by the package that owns the concept.

```xml
<!-- objectTypeDefinition.xml -->
<data xmlns="http://www.woltlab.com" …>
    <import>
        <definition>
            <name>com.example.mypkg.someFeature</name>
            <!-- optional interface all implementing object types must fulfil -->
            <interfacename>com\example\mypkg\system\objecttype\ISomeFeatureObjectType</interfacename>
        </definition>
    </import>
</data>
```

### Object Type

An *object type* implements an existing definition, binding your code into the core's extensibility mechanism.

```xml
<!-- objectType.xml -->
<data xmlns="http://www.woltlab.com" …>
    <import>
        <type>
            <name>com.example.mypkg.user.activityPoint</name>
            <definitionname>com.woltlab.wcf.user.activityPoint</definitionname>
            <classname>com\example\mypkg\system\user\activity\point\MyActivityPointObjectType</classname>
        </type>
    </import>
</data>
```

---

## 4. Template Listeners

Template listeners allow injecting Smarty template snippets into specific locations in core (or third-party) templates without overriding them.

```xml
<!-- templateListener.xml -->
<data xmlns="http://www.woltlab.com" …>
    <import>
        <templatelistener>
            <name>myUserProfileAddition</name>
            <templatename>userProfile</templatename>
            <eventname>afterContent</eventname>
            <!-- Template file at: templates/userProfileMyAddition.tpl -->
            <templatecode><![CDATA[{include file='userProfileMyAddition'}]]></templatecode>
            <environment>user</environment>
        </templatelistener>
    </import>
</data>
```
