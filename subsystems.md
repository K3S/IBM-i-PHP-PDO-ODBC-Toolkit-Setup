---
title: Running in Custom Subsystems
nav_order: 7
---

# Running ODBC and Apache in Custom Subsystems

By default, ODBC traffic on IBM i runs in the `QUSRWRK` subsystem, alongside FTP, file shares, and a long list of other system services. Apache instances run in `QHTTPSVR`. For most shops this is fine. For shops that want isolation — separate priorities, separate memory pools, separate monitoring, the ability to end one workload without touching the others — moving each into its own subsystem is the right answer.

This chapter walks through both. The patterns are essentially the same; only the entry points differ.

---

## Why isolate at all?

Three concrete reasons, in order of how often they matter:

1. **Performance isolation.** A runaway PHP request shouldn't be able to starve FTP transfers of CPU, and vice versa. Separate subsystems with separate run priorities and memory pools enforce this.
2. **Operational clarity.** When you `WRKACTJOB SBS(YOURSBS)`, you see only the jobs you care about. When something is wrong with web traffic, you can end and restart just that subsystem.
3. **Routing flexibility.** Once you have a custom subsystem, routing rules let you send specific user profiles or specific IP addresses to it — useful for separating customer-facing traffic from internal traffic, or development from production.

The cost is a small amount of one-time setup and one extra concept to remember when you're troubleshooting.

---

## ODBC in its own subsystem

The pattern below is based on IBM's [Creating a user-defined subsystem](https://www.ibm.com/docs/en/i/7.5?topic=subsystem-creating-user-defined-server) and Dawn May's [Routing ODBC Requests to a User-Defined Subsystem](https://dawnmayi.com/2015/08/03/route-db2-requests-to-a-specific-subsystem/). Both are recommended reading.

### Step 1 — Create a library to hold the subsystem objects

```
CRTLIB ODBCLIB TEXT('Subsystem objects for custom ODBC subsystem')
```

The library name is your choice. We use `ODBCLIB`; substitute whatever fits your conventions.

### Step 2 — Create a class

The class defines the run priority, time slice, and default wait for jobs running in the subsystem.

```
CRTCLS CLS(ODBCLIB/ODBCSBS)
       RUNPTY(20)
       TIMESLICE(2000)
       DFTWAIT(30)
       TEXT('Class for custom ODBC subsystem')
```

Adjust `RUNPTY` to match how aggressively you want ODBC to compete with other workloads — 20 is a common middle-ground priority for interactive web traffic.

### Step 3 — Create the subsystem description

Two options. Either start from scratch:

```
CRTSBSD SBSD(ODBCLIB/ODBCSBS)
        POOLS((1 *BASE))
        TEXT('Custom subsystem for ODBC traffic')
```

Or duplicate `QUSRWRK`, which inherits its memory pool layout and routing entries:

```
CRTDUPOBJ OBJ(QUSRWRK)
          FROMLIB(QSYS)
          OBJTYPE(*SBSD)
          TOLIB(ODBCLIB)
```

For most shops, duplicating `QUSRWRK` is the safer starting point — it gets you the same defaults the system uses successfully, and you tune from there.

### Step 4 — Add prestart job entries for QZDASOINIT

This is the step most online guides omit. Without it, the subsystem starts but no ODBC jobs ever land in it.

```
ADDPJE SBSD(ODBCLIB/ODBCSBS)
       PGM(QSYS/QZDASOINIT)
       INLJOBS(50)
       THRESHOLD(4)
       JOB(QZDASOINIT)
       STRJOBS(*YES)
```

`INLJOBS(50)` tells the subsystem to start 50 prestart jobs when it starts up. `THRESHOLD(4)` keeps the available pool from dropping below four idle jobs by spinning up new ones as activity rises. Tune both to match your peak traffic.

### Step 5 — Route specific user profiles to the subsystem

The routing call uses SQL, so run it from ACS Run SQL Scripts or any SQL tool — not from a 5250 command line.

```sql
CALL QSYS2.SET_SERVER_SBS_ROUTING('ODBCUSER', 'QZDASOINIT', 'ODBCSBS');
```

This routes all ODBC connections authenticating as `ODBCUSER` to the `ODBCSBS` subsystem. Repeat for every user profile that should land there.

To remove a routing entry, set the subsystem to `NULL`:

```sql
CALL QSYS2.SET_SERVER_SBS_ROUTING('ODBCUSER', 'QZDASOINIT', NULL);
```

{: .warning }
> The user profile being routed needs the same authority and library-list configuration as your default ODBC user. A common bug is routing a profile that doesn't have authority to the application's libraries — connections succeed but every query fails.

### Step 6 — Verify

Start the subsystem (`STRSBS ODBCLIB/ODBCSBS`), trigger a request from your PHP application, and run `WRKACTJOB SBS(ODBCSBS)`. You should see `QZDASOINIT` jobs landing in the new subsystem instead of `QUSRWRK`.

---

## Apache in its own subsystem

The pattern is similar. The reference is Dawn May's [Run an HTTP Server in its own subsystem](https://dawnmayi.com/2014/05/21/run-an-http-server-in-its-own-subsystem/).

### Step 1 — Create the subsystem

Same library/class/subsystem-description pattern as above:

```
CRTLIB HTTPLIB TEXT('Subsystem objects for custom Apache subsystem')

CRTCLS CLS(HTTPLIB/HTTPSBS)
       RUNPTY(25)
       TIMESLICE(2000)
       DFTWAIT(30)
       TEXT('Class for custom Apache subsystem')

CRTDUPOBJ OBJ(QHTTPSVR)
          FROMLIB(QSYS)
          OBJTYPE(*SBSD)
          TOLIB(HTTPLIB)
```

### Step 2 — Update the Apache configuration

In your Apache configuration file (typically `/www/<server-name>/conf/httpd.conf`), set the subsystem to use:

```apache
HTTPSubsystem HTTPLIB/HTTPSBS
```

The directive tells Apache which subsystem to start its workers in. Apache itself can be started from any context; the workers go where this directive says.

### Step 3 — Restart Apache

End the current Apache instance and restart it:

```
ENDTCPSVR SERVER(*HTTP) HTTPSVR(yourservername)
STRTCPSVR SERVER(*HTTP) HTTPSVR(yourservername)
```

Verify with `WRKACTJOB SBS(HTTPSBS)` — you should see your Apache `QHTTPSVR` jobs running there.

---

## A note on NGINX subsystems

NGINX is not a native IBM i workload, so it doesn't get routed by `HTTPSubsystem` directives. If you started NGINX from `QSH` or SSH, it runs in whatever subsystem your interactive session is in — usually `QINTER`. To run it in a dedicated subsystem, the cleanest path is to wrap the NGINX startup in a CL program submitted with `SBMJOB`, targeting the subsystem you've created.

For most shops, the operational benefit of moving NGINX into its own subsystem is small enough that it's not worth the effort — the bigger wins are with ODBC and the underlying `QZDASOINIT` jobs, which are where load actually concentrates.

---

## Where next

- **[Performance](performance.html)** — additional levers worth knowing about: prestart job tuning, exit programs, traces, and the things that materially affect ODBC and Apache latency in production.
- **[Reference](reference.html)** — connection string keywords and acknowledgments.
