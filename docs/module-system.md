# Module System

WoltLab Suite Core is itself split into a **core** package (`com.woltlab.wcf`) and several first-party application packages (WoltLab Suite Forum, Blog, Gallery, …). Third-party developers can build their own *application* packages using exactly the same mechanism. This document explains how to build a self-contained module — such as a forum, a file database, or a project tracker — that can be installed and used alongside WCS.

---

## 1. Package vs. Application

| Concept | Description |
|---|---|
| **Plugin / Package** | Extends or modifies an existing application. Does not run stand-alone. |
| **Application** | A self-contained module with its own URL base (e.g. `/forum/`), controllers, templates, and database tables. |

Set `<isapplication>1</isapplication>` in your `package.xml` to make your package an application.

---

## 2. Application Class

Every WCS application must provide an *application class* that extends `wcf\system\application\AbstractApplication`:

```php
<?php

namespace com\example\mymodule\system;

use wcf\system\application\AbstractApplication;

final class MyModuleCore extends AbstractApplication
{
    /**
     * Short abbreviation used as PHP namespace prefix and URL segment.
     * Must be unique across all installed packages.
     */
    protected $abbreviation = 'mymodule';

    /**
     * Fully-qualified class name of the default front-end controller.
     */
    protected $primaryController = 'com\example\mymodule\page\IndexPage';
}
```

Register it as a `coreObject` in your `package.xml`:

```xml
<instruction type="coreObject" />
```

Backed by `coreObject.xml`:

```xml
<data xmlns="http://www.woltlab.com" …>
    <import>
        <object>
            <objectname>com\example\mymodule\system\MyModuleCore</objectname>
        </object>
    </import>
</data>
```

---

## 3. Directory Layout

WCS expects a specific directory layout on disk. When your package is installed with `<isapplication>1</isapplication>` and `<applicationdirectory>mymodule</applicationdirectory>`, WCS creates a sub-directory next to the core installation:

```
/var/www/html/
├── wcf/          ← WoltLab Suite Core
│   └── …
└── mymodule/     ← your application
    ├── index.php
    ├── acp/
    ├── lib/
    │   ├── action/
    │   ├── form/
    │   ├── page/
    │   └── system/
    └── templates/
```

Use the `file` PIP to install files into this directory by placing them under `files/` in your `.tar` archive.

---

## 4. Controllers

WCS uses a **Page / Form / Action** controller pattern:

| Base class | Purpose |
|---|---|
| `wcf\page\AbstractPage` | Read-only pages (GET) |
| `wcf\form\AbstractForm` | Pages with form submission (POST) |
| `wcf\action\AbstractAction` | AJAX / REST actions |
| `wcf\action\AbstractSecureAction` | AJAX actions requiring a CSRF token |

### Example page

```php
<?php

namespace com\example\mymodule\page;

use wcf\page\AbstractPage;

final class IndexPage extends AbstractPage
{
    public string $templateName = 'index'; // templates/index.tpl

    public array $items = [];

    public function readData(): void
    {
        parent::readData();
        // populate $this->items from the database
    }

    public function assignVariables(): void
    {
        parent::assignVariables();
        \wcf\system\WCF::getTPL()->assign('items', $this->items);
    }
}
```

Register pages via the `page` PIP (see [Package Development](package-development.md)).

---

## 5. Database Layer

### Migrations

Database schema changes are managed through PHP migration scripts executed by the `database` PIP:

```xml
<instruction type="database">acp/update_com.example.mymodule_1.0.0.php</instruction>
```

The migration script uses the `DatabaseTableBuilder` API:

```php
<?php

use wcf\system\database\table\DatabaseTableChangeProcessor;
use wcf\system\database\table\PartialDatabaseTable;
use wcf\system\database\table\column\IntDatabaseTableColumn;
use wcf\system\database\table\column\VarcharDatabaseTableColumn;
use wcf\system\database\table\index\DatabaseTablePrimaryIndex;

$tables = [
    PartialDatabaseTable::create('mymodule1_item')
        ->columns([
            IntDatabaseTableColumn::create('itemID')->length(10)->notNull()->autoIncrement(),
            VarcharDatabaseTableColumn::create('title')->length(255)->notNull(),
        ])
        ->indices([
            DatabaseTablePrimaryIndex::create()->columns(['itemID']),
        ]),
];

(new DatabaseTableChangeProcessor(
    /** @var \wcf\data\package\Package $this->installation->getPackage() */ $this->installation->getPackage(),
    $tables,
    \wcf\system\WCF::getDB()->getEditor()
))->process();
```

### Data Objects (ORM)

WCS ships a simple ORM built around `DatabaseObject` / `DatabaseObjectEditor` / `DatabaseObjectList`:

```php
<?php

namespace com\example\mymodule\data\item;

use wcf\data\DatabaseObject;

/**
 * @property-read int    $itemID
 * @property-read string $title
 */
final class Item extends DatabaseObject
{
    protected static $databaseTableName = 'mymodule1_item';
    protected static $databaseTableIndexName = 'itemID';
}
```

---

## 6. Templates

Templates are Smarty files stored under `templates/` and installed via the `template` PIP.

```smarty
{* templates/index.tpl *}
{include file='header'}

<div class="section">
    {foreach from=$items item=item}
        <p>{$item->title|encodeHTML}</p>
    {/foreach}
</div>

{include file='footer'}
```

---

## 7. Permissions (User Group Options)

Register per-group permissions in `userGroupOption.xml`:

```xml
<categoryname>mymodule.user</categoryname>
<optionname>canViewItems</optionname>
<optiontype>boolean</optiontype>
<defaultvalue>1</defaultvalue>
```

Check them at runtime:

```php
if (!WCF::getSession()->getPermission('user.mymodule.canViewItems')) {
    throw new PermissionDeniedException();
}
```

---

## 8. ACP Integration

Add entries to the ACP navigation tree via the `acpMenu` PIP and register ACP pages the same way as front-end pages (using the `page` PIP with `<controller>…AcpPage</controller>`).

---

## 9. Further Reading

* [Package Development](package-development.md) – how to package everything into an installable `.tar` archive
* [Plugin API](plugin-api.md) – extend the module with event listeners and object types
* [WoltLab Developer Docs](https://docs.woltlab.com/) – official upstream documentation
