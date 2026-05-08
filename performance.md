---
title: Performance Tuning
nav_order: 8
---

# Performance Tuning

A handful of operations-side levers materially affect PHP and ODBC performance on IBM i. None of them are about your application code; they're about how the platform schedules, prestarts, traces, and audits the underlying jobs.

This chapter is short by design. It's a checklist of things to verify, in order of how much they typically matter.

---

## 1. Prestart job tuning for QZDASOINIT

ODBC connections from PHP land in `QZDASOINIT` jobs. By default, IBM i prestarts a small pool of these and grows it on demand. If your PHP application opens connections faster than the subsystem can spin up new prestart jobs, requests queue waiting for a job to become available — visible as latency spikes that don't correlate with CPU load.

The relevant attributes are `INLJOBS` (initial pool size) and `THRESHOLD` (the floor at which new jobs are spun up).

To inspect the current configuration:

```
DSPSBSD SBSD(QSYS/QUSRWRK)
```

Then choose option 10 (Prestart job entries) and find the entry for `QZDASOINIT`.

To change it:

```
CHGPJE SBSD(QSYS/QUSRWRK)
       PGM(QSYS/QZDASOINIT)
       INLJOBS(50)
       THRESHOLD(10)
       MAXJOBS(*NOMAX)
```

If you've moved ODBC into a custom subsystem (see [Subsystems](subsystems.html)), apply these settings there instead.

The right values depend entirely on your traffic shape. Start with `INLJOBS(50)` and `THRESHOLD(10)`, watch with `WRKACTJOB SBS(...)` under realistic load, and adjust until the active job count comfortably exceeds your concurrent connection count.

For IBM's full guidance: [IBM i Database Host Server and the QZDASOINIT Prestart Jobs](https://www.ibm.com/support/pages/ibm-i-database-host-server-and-qzdasoinit-prestart-jobs).

---

## 2. Audit your exit programs

Exit programs are user-written hooks that fire on specific system events — including every ODBC connection attempt and every database query through ODBC. They're powerful: you can use them for fine-grained authorization, audit logging, or query rewriting.

They're also a common cause of latency that's invisible from the application side. An exit program that runs `O(1)` per query but does 50ms of work adds 50ms to every query. Multiply by your query volume and the cost is significant.

To find exit programs registered for ODBC:

```
WRKREGINF EXITPNT(QIBM_QZDA_*)
```

Inspect every entry. For each one, ask:

- Is it actually doing something the application needs?
- If yes, is the implementation efficient — particularly the database access pattern?
- If no, can we remove it?

For background, see [Harnessing Your ODBC Users with Exit Programs](https://www.itjungle.com/2006/11/29/fhg112906-story02/).

{: .warning }
> Don't remove exit programs casually. They're often there for a security reason that someone made years ago and never documented. Audit them, understand what they do, then decide.

---

## 3. Disable TCP/IP application traces in production

`TRCTCPAPP` is a powerful diagnostic tool that captures detailed traffic for a specific TCP/IP application. The cost is significant — traces add real overhead per connection, and a forgotten trace can sit on a system for months adding latency to every request.

To check:

```
TRCTCPAPP APP(*ALL) SET(*CHK)
```

If anything is enabled and you don't actively need the data, end it:

```
TRCTCPAPP APP(*ALL) SET(*OFF)
```

For specific applications:

```
TRCTCPAPP APP(*HTTP) SET(*OFF)
TRCTCPAPP APP(*ODBC) SET(*OFF)
```

Reference: [Trace TCP/IP Application (TRCTCPAPP)](https://www.ibm.com/docs/en/i/7.5?topic=ssw_ibm_i_75/cl/trctcpapp.htm).

---

## 4. Right-size your PHP FastCGI children

The number of PHP worker processes is a deliberate concurrency cap. Set it too low and requests queue at the web tier. Set it too high and you spend CPU on context switches and waste memory on idle workers.

For Apache + FastCGI, the setting is `PHP_FCGI_CHILDREN` in `fastcgi.conf` (see [Apache + PHP](apache.html)).

For NGINX + PHP-FPM, the equivalents are `pm.max_children`, `pm.start_servers`, `pm.min_spare_servers`, and `pm.max_spare_servers` in `/QOpenSys/etc/php/php-fpm.d/www.conf`.

Two articles cover sizing on IBM i specifically and are worth the read:

- [Guru: Right Size Your PHP](https://www.itjungle.com/2018/12/03/guru-right-size-your-php/)
- [Optimize your IBM i web application using FastCGI](https://www.seidengroup.com/2021/10/28/optimize-ibmi-web-application-fastcgi/)

The general rule: start with what your peak concurrent request count actually is (measured, not guessed), add headroom, then monitor and adjust.

---

## 5. Cycle FPM workers periodically

PHP processes accumulate memory over time — not always due to leaks in your code; sometimes from extension internals, sometimes from cached opcache state, sometimes from connection state on the database side that doesn't reset cleanly between requests.

Set `pm.max_requests` in `www.conf` to a reasonable number (500-2000 is typical) so each worker recycles after that many requests. The slight cost of restarting a worker periodically is well worth the bounded memory footprint and the fact that any leaked connection state gets cleared along with the worker.

```ini
pm.max_requests = 1000
```

---

## 6. Use forward-only cursors for all ODBC reads

Already covered in [Connecting to DB2](connecting-to-db2.html#the-one-phpini-setting-that-matters), but worth restating here because it's the highest-leverage `php.ini` setting on the list:

```ini
odbc.default_cursortype=0
```

For any web application that doesn't scroll backwards through result sets (which is essentially all of them), this is free performance.

---

## 7. Verify you're using persistent connections

For a typical web request, opening a fresh ODBC connection takes longer than the actual query. Persistent connections (`PDO::ATTR_PERSISTENT => true`) reuse the underlying ODBC handle across requests within the same FPM/FastCGI worker — you trade some additional discipline around connection state for a substantial latency improvement.

If you're unsure whether your stack is using persistent connections, watch `WRKACTJOB SBS(QUSRWRK)` (or your custom subsystem) under steady traffic. With persistent connections, the `QZDASOINIT` job count stays roughly flat at your worker count. Without them, it churns up and down with every request.

---

## A summary checklist

The seven items above, in checklist form:

- [ ] `QZDASOINIT` prestart jobs sized for peak traffic
- [ ] Exit programs audited; unused ones removed
- [ ] No `TRCTCPAPP` traces left running in production
- [ ] PHP FastCGI / FPM children sized to actual peak concurrency
- [ ] FPM `pm.max_requests` set to recycle workers periodically
- [ ] `odbc.default_cursortype=0` set in `php.ini`
- [ ] PDO `ATTR_PERSISTENT => true` confirmed in production

Most shops will find that walking this list once a year is enough to keep performance from drifting.

---

## Where next

- **[Reference](reference.html)** — connection string keywords, the IBM repos, and acknowledgments to the people whose work this guide builds on.
