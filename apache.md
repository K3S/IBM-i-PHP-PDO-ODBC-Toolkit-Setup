---
title: Apache + PHP
nav_order: 3
---

# Running PHP under Apache on IBM i

This is IBM's traditional path: the bundled IBM HTTP Server (Apache) handles the request, hands `.php` files off to a FastCGI process group running `php-cgi`, and returns the result. It's well-trodden, configurable through a web interface, and what most existing IBM i shops are running today.

If you don't already have a strong reason to choose NGINX, Apache is the lower-friction starting point.

---

## 1. Create the Apache instance

Open the IBM HTTP Admin interface in a browser:

```
http://YOUR-IBM-I-IP:2001/HTTPAdmin
```

{: .note }
> The HTTP Admin interface only loads if the **Admin Server** is running on the partition. If the page won't load, an admin needs to start it. See IBM's [Manage server documentation](https://www.ibm.com/docs/en/i/7.5?topic=server-managing-http-administration-server) for the command.

Click **Create HTTP Server** in the top left, and walk through the wizard. Note the web-root paths it creates — typically `/www/<server-name>/htdocs` for the document root and `/www/<server-name>/conf` for the configuration. The rest of this chapter assumes the defaults; adjust accordingly if you customized them.

---

## 2. Configure Apache to run PHP

Use the **Edit Configuration File** option in the left panel of HTTP Admin and add the following near the top of the config:

```apache
# Load Apache's FastCGI support (originally created for Zend)
LoadModule zend_enabler_module /QSYS.LIB/QHTTPSVR.LIB/QZFAST.SRVPGM

# Tell Apache that any .php file should be handled by FastCGI
AddType    application/x-httpd-php .php
AddHandler fastcgi-script .php

# Allow URLs without /index.php
DirectoryIndex index.php index.html
```

Click **OK** at the bottom of the page to save.

---

## 3. Configure FastCGI

Apache now knows to route `.php` files to a FastCGI handler — but the handler itself isn't configured yet. Without this step, your browser will simply download the PHP source instead of executing it.

Create a file named `fastcgi.conf` in `/www/<server-name>/conf` with the following contents:

```apache
Server type="application/x-httpd-php" \
       CommandLine="/QOpenSys/pkgs/bin/php-cgi" \
       StartProcesses="1" \
       SetEnv="PHP_FCGI_CHILDREN=10"
```

`PHP_FCGI_CHILDREN=10` starts ten PHP worker processes. Each handles one request at a time, so this is effectively your concurrency cap. Increase it if you start seeing requests queue under load (more on tuning below).

Start the web server, either through the HTTP Admin UI or with:

```
STRTCPSVR SERVER(*HTTP) HTTPSVR(yourservername)
```

---

## 4. Test

Drop an `index.php` into your web root containing:

```php
<?php phpinfo(); ?>
```

Visit the virtual host you set up. If you see the PHP info page, you're running PHP via the RPMs on Apache. If you instead get a download prompt or a 404, double-check the `LoadModule`, `AddType`, and `fastcgi.conf` steps above — that combination is where almost every problem lives.

---

## 5. Recommended additional configuration

The defaults work, but a few additions materially improve performance and prevent a common encoding issue. Add these to your Apache config:

```apache
# gzip compression — keep these near the top with the other LoadModules
LoadModule deflate_module /QSYS.LIB/QHTTPSVR.LIB/QZSRCORE.SRVPGM
AddOutputFilterByType DEFLATE \
    application/x-httpd-php \
    application/json \
    text/css \
    application/x-javascript \
    application/javascript \
    text/html

# Keep-alive and connection tuning
TimeOut          30000
KeepAliveTimeout 30
HotBackup        Off
ThreadsPerChild  40

# CCSID — fixes the "my output looks scrambled" problem most shops hit eventually
DefaultFsCCSID   37
CGIJobCCSID      37
```

The `DefaultFsCCSID` and `CGIJobCCSID` lines deserve a callout: when output looks like high-ASCII garbage, the cause is almost always a CCSID mismatch between the job running PHP and the data it's reading. Setting both to 37 (the standard EBCDIC CCSID for U.S. English) resolves the most common version of this.

{: .tip }
> If your shop runs on a non-U.S. CCSID (e.g., 273, 285, 297, 1141), substitute that value instead of 37. The point is consistency between the filesystem CCSID, the job CCSID, and what your DB2 tables expect.

---

## Tuning Apache and PHP for production

Getting PHP serving requests is one milestone. Sizing it correctly under load is another. Two articles are worth the read before you push to production:

- [Guru: Right Size Your PHP](https://www.itjungle.com/2018/12/03/guru-right-size-your-php/) — covers sizing FastCGI children against memory and CPU on IBM i.
- [Optimize your IBM i web application using FastCGI](https://www.seidengroup.com/2021/10/28/optimize-ibmi-web-application-fastcgi/) — Seiden's deep dive on FastCGI tuning specifically.

The short version: start with `PHP_FCGI_CHILDREN=10`, watch your QHTTPSVR job memory and active request count under realistic load, and increase incrementally until p95 latency stops improving. Don't blindly set this to 100; each child is a real PHP process with real memory cost.

---

## Automating the Apache setup

Seiden Group publishes a tool, **siteadd**, that wraps the steps above into a single command:

[Automatically configure IBM i Apache web server to run PHP with siteadd](https://www.seidengroup.com/php-documentation/automatically-configure-ibm-i-apache-web-server-to-run-php-with-siteadd/)

If you're standing up multiple instances or scripting environment setup, it's worth using.

---

## Where next

- **[Connecting to DB2](connecting-to-db2.html)** — once Apache + PHP is up, the next question is how to talk to your database.
- **[Running Apache in its own subsystem](subsystems.html#apache-in-its-own-subsystem)** — for shops that want to isolate web traffic from QUSRWRK.
- **[Performance tuning](performance.html)** — prestart jobs, exit programs, and traces that affect latency.
