---
title: Calling RPG from PHP
nav_order: 6
---

# Calling RPG from PHP via the Toolkit

You have an ODBC connection to DB2. You can run queries. The next step — and the one that makes the IBM i platform genuinely powerful when paired with PHP — is calling RPG and CL programs from PHP using that same connection.

The mechanism is the **[IBM i PHP Toolkit](https://github.com/zendtech/IbmiToolkit)**, originally developed by Zend (now maintained by Perforce) on top of XMLSERVICE. The Toolkit handles parameter marshalling, calls XMLSERVICE on the IBM i, parses the response, and gives you a PHP array back.

The interesting part for cross-platform shops: the Toolkit can use your existing PDO connection as its transport, regardless of whether PHP is running on the IBM i or on a separate Linux/Windows server. One connection pool. One set of credentials. Two protocols on the same wire.

---

## Why this pattern matters

We mentioned in the [overview](./) that K3S follows a CQRS split — RPG owns writes through single-purpose APIs, PHP owns reads. The Toolkit is what makes that split practical.

When the PHP layer needs to create a supplier, it doesn't write to the supplier table directly. It calls an `ADDSUPL` RPG program, which validates the input against business rules that live in one place, performs the write, and returns either success or a structured error message. The same RPG program is callable from a 5250 menu, from a batch job, or from a CL routine — and now also from PHP.

This means the PHP code never carries duplicated business logic, and a future re-platform of the front end never requires re-implementing the rules.

---

## The Toolkit's modes of operation

The Toolkit can connect to the IBM i three ways:

1. **`db2`** — uses the native `ibm_db2` PHP extension. Fastest. Requires PHP to be running on IBM i (or on Linux with `ibm_db2` and the proper drivers, which is uncommon).
2. **`odbc`** — opens a fresh ODBC connection of its own.
3. **`pdo`** — reuses an existing PDO connection you've already opened.

The third mode is the interesting one. Whatever connection your PDO adapter is using — including a persistent one — gets handed to the Toolkit, which marshals XMLSERVICE calls over the same wire. No second login, no second connection in your pool.

---

## Connecting the Toolkit to your PDO connection

The example below is from a Laminas/Mezzio-style service factory; the pattern translates directly to Symfony, Laravel, or plain PHP. The key idea: get the underlying ODBC resource out of the PDO adapter, then pass it to the Toolkit constructor.

```php
<?php

namespace RPG\Service\Factory;

use Psr\Container\ContainerInterface;
use Laminas\Db\Adapter\AdapterInterface;
use ToolkitApi\Toolkit;

class ToolkitFactory
{
    public function __invoke(ContainerInterface $container): Toolkit
    {
        /** @var AdapterInterface $adapter */
        $adapter = $container->get(AdapterInterface::class);

        // Reach through the adapter to the underlying ODBC resource
        $dbConn = $adapter
            ->getDriver()
            ->getConnection()
            ->getResource();

        // The fourth argument tells the Toolkit to use the existing
        // resource as a PDO/ODBC connection rather than opening its own
        return new Toolkit($dbConn, null, null, 'pdo');
    }
}
```

Once instantiated, the Toolkit calls programs like this:

```php
$params = [
    $toolkit->AddParameterChar('both', 10, 'inSuplCode', 'inSuplCode', 'ACME001'),
    $toolkit->AddParameterChar('both', 50, 'outName',    'outName',    ''),
    $toolkit->AddParameterChar('both',  1, 'outActive',  'outActive',  ''),
];

$result = $toolkit->PgmCall('GETSUPL', 'K3SLIB', $params);

echo trim($result['io_param']['outName']);
```

For complete Toolkit examples covering data structures, arrays, return values, and CL command execution, see the [Toolkit documentation](https://github.com/zendtech/IbmiToolkit/blob/master/README.md) and the older but still useful [Toolkit sample scripts reference](https://docs.roguewave.com/en/zend/Zend-Server-7-IBMi/content/toolkit_sample_scripts.htm).

---

## What this looks like in practice

The end-to-end flow for a "create a supplier" request from a React frontend talking to a Mezzio backend talking to RPG on IBM i:

1. React posts a JSON form payload to the Mezzio API.
2. Mezzio validates the shape, then asks the container for the Toolkit service.
3. The Toolkit factory pulls the underlying ODBC connection out of the existing PDO adapter and constructs the Toolkit with `'pdo'` mode.
4. Mezzio calls `$toolkit->PgmCall('ADDSUPL', $library, $params)`.
5. The Toolkit serializes the parameters into XMLSERVICE XML and writes it over the existing ODBC connection.
6. XMLSERVICE on IBM i deserializes, calls `ADDSUPL`, and returns the result as XML.
7. The Toolkit parses the response into a PHP array.
8. Mezzio returns JSON to React.

There's no second connection, no second authentication, no separate pool to manage. The whole thing rides on the connection PHP was already using to query the database.

---

## A few practical notes

### Stateless vs. stateful jobs

The Toolkit can run in **stateless** mode (every call starts a fresh XMLSERVICE job) or **stateful** mode (XMLSERVICE jobs are kept alive between calls, identified by a session token).

- **Stateless** is simpler and avoids the cleanup problem entirely. The cost is a job start per call, which is non-trivial.
- **Stateful** is much faster for high-volume call patterns, but you take on responsibility for cleaning up orphaned jobs.

For most web request workloads, stateless is the right default. Switch to stateful when you're sure you need it and you've planned the lifecycle.

### Parameter sizing

The number-one cause of mysterious Toolkit failures is parameter length mismatch. If your RPG declares a parameter as `char(10)` and PHP sends a 12-character value, XMLSERVICE will either truncate silently or fail in a way that's hard to diagnose. Always be explicit, always match exactly.

This is a place where well-defined service program prototypes pay for themselves — they're the contract that tells both sides what to expect.

### CCSID 65535 source members

If your RPG source members or include files were created decades ago with no CCSID set, they may default to CCSID 65535 (binary, no translation). PHP cannot read XMLSERVICE responses correctly when this happens. The fix is on the IBM i side: set CCSID 37 (or your shop's standard) on the affected source members and any tables they reference.

---

## Releasing locks from Toolkit calls

Toolkit calls run inside the same `QZDASOINIT` jobs that handle ODBC queries, so the same lock-release guidance from [Connecting to DB2](connecting-to-db2.html#releasing-locks-held-by-odbc-jobs) applies directly. Nothing additional is needed.

---

## Where next

- **[Subsystems](subsystems.html)** — for shops that want to isolate ODBC and Apache traffic from QUSRWRK, including the prestart-job configuration that affects Toolkit performance.
- **[Performance](performance.html)** — exit programs, traces, and prestart settings that materially affect Toolkit latency.
- **[RPG Tutorial](https://rpgtutorial.k3s.com)** — if you want to write the RPG side of this pattern from scratch, our companion tutorial covers free-format RPGLE, service programs, and the Toolkit-call interface conventions.
