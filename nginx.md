---
title: NGINX + PHP-FPM
nav_order: 4
---

# Running PHP under NGINX on IBM i

NGINX is the modern alternative to Apache for IBM i PHP deployments. It performs noticeably better under concurrent load, configuration is more legible, and it's the standard outside the IBM i world for new PHP work. The trade-off is that there's no IBM-supplied admin UI — everything is hand-edited config files.

NGINX has a built-in FastCGI client, but no FastCGI process manager. For that, we use the standard PHP-FPM utility, which the Zend RPMs include.

The setup pattern below mirrors the [NGINX team's WordPress recipe](https://www.nginx.com/resources/wiki/start/topics/recipes/wordpress/), adapted for IBM i conventions.

---

## 1. Install NGINX

From the OSPM, install the `nginx` package. After installation, NGINX lives at `/QOpenSys/pkgs/sbin/nginx` and reads config from `/QOpenSys/etc/nginx/`.

---

## 2. Lay out the directory structure

We'll mirror the IBM i Apache convention so the directory layout feels familiar:

| Path | Purpose |
|------|---------|
| `/www/php` | Server base |
| `/www/php/logs` | NGINX access and error logs |
| `/www/php/htdocs` | Document root — your `.php` files go here |

Create those directories before starting the server.

---

## 3. Write the NGINX configuration

Create `/QOpenSys/etc/nginx/php.conf`:

```nginx
error_log  /www/php/logs/nginx.err.log;
pid        /www/php/logs/nginx.pid;

events {
    # required section, leave empty for defaults
}

http {
    upstream php {
        # default FPM listen address
        server 127.0.0.1:9000;
    }

    server {
        # set your actual hostname here
        server_name mywebserver.example.com;

        # listen port for this site
        listen 6090 default_server;

        # document root
        root /www/php/htdocs;

        # allow URLs without /index.php
        index index.php index.html;

        # case-sensitive .php match — use ~* for case-insensitive
        location ~ \.php$ {
            include /QOpenSys/etc/nginx/snippets/fastcgi-php.conf;
            fastcgi_pass php;
        }
    }
}
```

Then create the FastCGI snippet referenced above at `/QOpenSys/etc/nginx/snippets/fastcgi-php.conf`:

```nginx
fastcgi_split_path_info ^(.+\.php)(/.+)$;

# Verify the script exists before passing it
try_files $fastcgi_script_name =404;

# Restore $fastcgi_path_info after try_files (a known NGINX quirk)
# https://trac.nginx.org/nginx/ticket/321
set $path_info $fastcgi_path_info;
fastcgi_param PATH_INFO $path_info;

fastcgi_index index.php;
include fastcgi.conf;
```

---

## 4. Configure PHP-FPM

PHP-FPM's main config file is `/QOpenSys/etc/php/php-fpm.conf`, which mostly just delegates to `/QOpenSys/etc/php/php-fpm.d/www.conf`. The defaults are fine for getting started — except for one thing.

FPM requires its worker processes to run under a real user *and* group. The default user is `QTMHHTTP`, but `QTMHHTTP` has no primary group. FPM detects this as an error:

```
[19-Jul-2019 13:27:35] ERROR: [pool www] please specify user and group other than root
[19-Jul-2019 13:27:35] ERROR: FPM initialization failed
```

You have two options:

1. **Give `QTMHHTTP` a primary group:**
   ```
   CHGUSRPRF USRPRF(QTMHHTTP) GRPPRF(GRP1)
   ```

2. **Override user and group in the FPM pool config**, by editing `/QOpenSys/etc/php/php-fpm.d/www.conf`:
   ```ini
   user  = qtmhhttp
   group = grp1
   ```

Either works. Pick the option that fits your shop's user-management conventions.

---

## 5. Start everything

```bash
# NGINX, using the config you just wrote
/QOpenSys/pkgs/bin/nginx -c php.conf

# PHP-FPM (runs in the background)
/QOpenSys/pkgs/sbin/php-fpm
```

NGINX runs under the user profile that started it; FPM runs under whatever user/group is configured in `www.conf`.

---

## 6. Test

Drop an `index.php` into `/www/php/htdocs`:

```php
<?php phpinfo(); ?>
```

Visit `http://your-host:6090/`. If you see the PHP info page, the stack is working.

---

## Why NGINX over Apache?

For new IBM i PHP deployments, the reasons to prefer NGINX are practical:

- **Better concurrency.** NGINX is event-driven; Apache (in the IBM i FastCGI configuration) is process-per-request. Under load, NGINX handles more requests per CPU.
- **Cleaner configuration.** The config file is a single readable document, not a wizard-generated mix of Apache directives.
- **First-class PHP-FPM integration.** FPM is the de facto standard process manager for PHP, and it's what every PHP performance article you'll read assumes.
- **Familiar to non-IBM-i developers.** A new hire who's never touched a 5250 will recognize an NGINX config immediately.

The reasons to stick with Apache:

- **You already have it running.** Existing Apache configs don't need to be migrated unless there's a problem to solve.
- **You rely on the HTTP Admin UI.** NGINX has no equivalent.
- **You're integrating with IBM-supplied modules** that depend on the IBM HTTP Server — these are rare but exist.

For a new PHP application on a new partition, NGINX is the right default.

---

## Where next

- **[Connecting to DB2](connecting-to-db2.html)** — the next question is how PHP talks to your database.
- **[Performance tuning](performance.html)** — prestart jobs, exit programs, and traces that affect latency regardless of which web server you chose.
