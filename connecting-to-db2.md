---
title: Connecting to DB2
nav_order: 5
---

# Connecting PHP to DB2 via PDO and ODBC

This is the chapter that this guide originally existed for: how to connect a PHP application to DB2 on IBM i using PDO and ODBC, whether PHP is running on the IBM i itself or on a separate Linux/Windows server.

---

## What PDO and ODBC are, briefly

**PDO** (PHP Data Objects) is PHP's database abstraction layer. You write your queries against the PDO API, and PDO handles the dialect translation and connection mechanics. The same query code runs against MySQL, PostgreSQL, DB2, or anything else with a PDO driver.

**ODBC** (Open Database Connectivity) is a vendor-neutral standard for talking to databases over a defined wire protocol. Every major database ships an ODBC driver.

The combination gives you portability on both axes: PDO abstracts the application code from the database vendor, and ODBC abstracts the connection from the platform. For IBM i specifically, this means a PHP application can run anywhere — IBM i, Linux, Windows, a container — and talk to DB2 on IBM i without any platform-specific code paths.

The IBM-supplied connection-string keyword reference is the canonical guide to what's available:
[IBM i ODBC Connection String Keywords](https://www.ibm.com/docs/en/i/7.5?topic=details-connection-string-keywords).

---

## The one `php.ini` setting that matters

Before any of the connection examples below, set this in your `php.ini` (or a `.user.ini` in your application's directory):

```ini
odbc.default_cursortype=0
```

This sets the cursor type to **forward-only**, which can produce dramatic speed improvements on result sets of any meaningful size. The default cursor type is bidirectional, which the ODBC driver implements by buffering results — fine for ten rows, painful for ten thousand.

See PHP's [ODBC configuration documentation](https://www.php.net/manual/en/odbc.configuration.php#ini.uodbc.defaultcursortype) for the full explanation.

{: .tip }
> If your app does only forward-only reads (which is true of almost every web application), there is no downside to this setting. Make it part of your default `php.ini`.

---

## Scenario A — PHP running on IBM i, connecting to DB2 on the same IBM i

The most common starting setup: a single partition runs both the web tier and the database.

### Prerequisites

You need two pieces installed in this order:

1. **`unixODBC` and `unixODBC-devel`** from OSPM. These provide the ODBC driver manager that PHP's `pdo_odbc` extension talks to.
2. **The IBM i Access ODBC Driver for PASE.** Download from IBM:
   [ODBC driver for the IBM i PASE environment](https://www.ibm.com/support/pages/odbc-driver-ibm-i-pase-environment).

{: .warning }
> Install order matters. `unixODBC` first, then the IBM ODBC driver. Reverse order causes the IBM installer to skip the registration steps that connect the driver to the manager, and you'll get cryptic "data source not found" errors at runtime.

### Connection example

The IBM documentation will mention configuring DSNs in `odbc.ini` or per-user `.odbc.ini`. You can do that, but for application code we strongly prefer DSN-less connection strings — they live in your repository, get diffed in pull requests, and don't require coordinating filesystem state across deploys.

```php
<?php
return [
    'db' => [
        'dsn' => 'odbc:DRIVER={IBM i Access ODBC Driver};'
               . 'SYSTEM=ipaddress;'
               . 'UID=ibmiusername;'
               . 'PWD=ibmipassword;'
               . 'NAM=1;'
               . 'TSFT=1;'
               . 'DBQ=, YOURLIB1 YOURLIB2 YOURLIB3',
        'driver' => 'Pdo',
        'platform' => 'IbmDb2',
        'platform_options' => [
            'quote_identifiers' => true,
        ],
        'driver_options' => [
            PDO::ATTR_PERSISTENT => true,
            PDO::ATTR_EMULATE_PREPARES => true,
        ],
    ],
];
```

### What the connection-string keywords do

| Keyword | Effect |
|---------|--------|
| `DRIVER` | Names the ODBC driver to use. Must match exactly what's registered in `odbcinst.ini`. |
| `SYSTEM` | The IBM i partition's IP or hostname. |
| `UID` / `PWD` | The IBM i user profile and password the connection authenticates as. |
| `NAM=1` | Use `*SYS` naming (slash-separated, IBM i-style) instead of SQL naming. Required if you use library-list-aware code. |
| `TSFT=1` | Return timestamps in IBM i format (`YYYY-MM-DD-HH.MM.SS.uuuuuu`). |
| `DBQ` | Library list. The leading comma sets the *default* library to none, then lists additional libraries to add to the list. |

{: .note }
> `DBQ` syntax confuses people more than any other keyword. Format: `default_library, lib1 lib2 lib3`. The default library goes before the comma; additional libraries follow, space-separated. To leave the default unset (almost always what you want), start with a leading comma.

---

## Scenario B — PHP running on Linux or Windows, connecting to DB2 on IBM i

This is the more interesting case for shops adopting modern PHP frameworks: the web tier moves off the IBM i (often to Linux containers behind a load balancer), but DB2 stays where it belongs.

The connection string is nearly identical — the only practical addition is the `CCSID` keyword, which avoids encoding surprises when crossing platforms.

```php
<?php
return [
    'db' => [
        'dsn' => 'odbc:DRIVER={IBM i Access ODBC Driver};'
               . 'SYSTEM=ipaddress;'
               . 'UID=ibmiusername;'
               . 'PWD=ibmipassword;'
               . 'NAM=1;'
               . 'CCSID=1208;'
               . 'DBQ=, YOURLIB1 YOURLIB2 YOURLIB3',
        'driver' => 'Pdo',
        'platform' => 'IbmDb2',
        'platform_options' => [
            'quote_identifiers' => true,
        ],
        'driver_options' => [
            PDO::ATTR_PERSISTENT => true,
            PDO::ATTR_EMULATE_PREPARES => true,
        ],
    ],
];
```

`CCSID=1208` requests UTF-8 for character data crossing the wire. This is what virtually every Linux/Windows PHP application wants.

### Prerequisites on the client side

- **Linux:** Install `unixODBC` from your distro's package manager, then install IBM's [Linux ODBC driver](https://www.ibm.com/support/pages/ibm-i-access-client-solutions). The driver registers itself with `unixODBC` automatically.
- **Windows:** Install IBM i Access Client Solutions, which installs the ODBC driver into the Windows ODBC manager. PHP for Windows uses the system ODBC manager directly — no `unixODBC` needed.

### Connection sanity check

A useful first step on a new client: prove you can connect at all using the OS-level tool, before bringing PHP into the picture.

On Linux:
```bash
isql -v "DRIVER={IBM i Access ODBC Driver};SYSTEM=10.0.0.5;UID=USER;PWD=PASS;NAM=1"
```

On Windows, use the **ODBC Data Source Administrator** to define a System DSN and click **Test Connection**.

If the OS-level test fails, the driver isn't installed correctly or the network path is blocked — fix that before debugging PHP.

---

## Persistent connections: when to use them

Both examples above use `PDO::ATTR_PERSISTENT => true`. This means PDO holds the underlying ODBC connection open across PHP requests in the same FastCGI/FPM child, reusing it instead of opening a fresh one.

The trade-offs:

- **For:** Connection setup is the slowest part of an ODBC call. Persistent connections make the hot path dramatically faster, particularly if your app issues a small number of queries per request.
- **Against:** Stuck connections, leftover library-list state, and held locks survive across requests until the FPM child recycles. You will eventually have a bad day caused by this.

For most shops, persistent + a sensible `pm.max_requests` setting in PHP-FPM (so workers cycle every few hundred requests) is the right balance. If you suspect connection state is leaking between requests, set `ATTR_PERSISTENT => false` while you debug; the slowdown is visible but tolerable.

---

## Releasing locks held by ODBC jobs

ODBC connections from PHP run as `QZDASOINIT` jobs on the IBM i. These jobs can hold record-level and member-level locks on files they've touched — particularly when the connection is persistent and a query was killed mid-execution.

Locks are released automatically when the job ends, which by default is 15-30 minutes after the last activity (or longer with the **Lazy Close** setting enabled).

For situations where you need to release locks immediately — most often during deployment or schema changes — K3S has open-sourced a small RPG utility that ends ODBC jobs cleanly:

[Release Locks From ODBC Connections](https://github.com/K3S/IBMi-Utilities/blob/master/ReleaseLocks/endjob.sqlrpgle)

Compile it, call it from CL with appropriate authority, and the held locks clear within seconds.

{: .caution }
> Don't run lock-release routines blindly against active production traffic. You'll kill in-flight requests. Schedule it during a quiet window or scope it to a specific user profile that no real users authenticate as.

---

## Where next

- **[Calling RPG from PHP](calling-rpg-from-php.html)** — once you have a connection, you can use it for more than just SQL. The IBM i Toolkit lets you reuse the same PDO connection to invoke RPG and CL programs.
- **[Performance tuning](performance.html)** — prestart `QZDASOINIT` jobs, exit programs, and the operations-side levers.
- **[Reference](reference.html)** — connection string keywords, the IBM repos, and acknowledgments.
