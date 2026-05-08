---
title: Installing PHP on IBM i
nav_order: 2
---

# Installing PHP on IBM i

There are two practical ways to get PHP onto an IBM i partition: install the **Zend PHP RPMs** through the Open Source Package Manager, or install **Zend Server** (the commercial distribution). For most shops, the RPMs are the right choice.

{: .note }
> The PHP RPMs and Zend Server are runtime-equivalent. The differences are in tooling and packaging: Zend Server adds debugging, deployment workflow, and a custom package manager, but does not include some extensions (notably `intl` and `zip`). The RPMs include those, are free, and are what the rest of this guide assumes.

---

## 1. Set up the Open Source Package Manager (OSPM)

OSPM is the standard way to install open-source software on modern IBM i. It comes bundled with IBM i ACS (Access Client Solutions). Follow IBM's instructions to enable it: [Getting started with Open Source Package Management](https://www.ibm.com/support/pages/getting-started-open-source-package-management-ibm-i-acs).

If SSH is not yet configured on your partition (which OSPM normally requires), see our companion guide: [How to set up OSPM without SSH](https://ospm.k3s.com).

Once OSPM is working, all open-source binaries live under `/QOpenSys/pkgs/`, and `yum` itself is at `/QOpenSys/pkgs/bin/yum`.

---

## 2. Install yum-utils

From the **Available Packages** tab in OSPM, install `yum-utils`. This gives you `yum-config-manager`, which you'll need to add the Zend repository.

---

## 3. Add the Zend PHP repository

From a shell (SSH preferred; `QSH` works in a pinch), add the Zend repo:

```bash
/QOpenSys/pkgs/bin/yum-config-manager --add-repo http://repos.zend.com/ibmiphp/
```

For the full list of community-maintained third-party repositories, see [3rd Party Open Source Repos for IBM i](https://ibmi-oss-docs.readthedocs.io/en/latest/yum/3RD_PARTY_REPOS.html).

{: .warning }
> Read the notes in the IBM OSS docs before you add the PHP repo. The Zend repo bundles a number of packages, and it's worth knowing what's in it before you install everything.

---

## 4. Install the PHP packages

Back in OSPM's **Available Packages** tab, install the packages whose names begin with `php`. The footprint is small, and you'll likely use most of them eventually. Pay particular attention to:

- `php` — the core runtime
- `php-cli` — command-line PHP
- `php-cgi` — the FastCGI binary used by Apache/NGINX
- `php-fpm` — FastCGI Process Manager (required for the NGINX path)
- `php-pdo`, `php-pdo-odbc` — PDO drivers
- `php-ibm_db2` — the native ibm_db2 extension (optional, but useful for some workloads)
- `php-mbstring`, `php-intl`, `php-zip`, `php-xml`, `php-curl` — common extensions Composer packages will expect

---

## 5. Configure PHP

The defaults are reasonable for most setups, but you'll want to know where the configuration lives.

The default `php.ini` ships at:

```
/QOpenSys/etc/php.ini
```

For PHP RPMs to load it, move (or copy) it into the `php` subdirectory:

```
/QOpenSys/etc/php/php.ini
```

Per-extension configuration lives in:

```
/QOpenSys/etc/php/conf.d
```

To verify your `php.ini` is being loaded, drop an `index.php` containing `<?php phpinfo(); ?>` into your web root and load it from a browser. The **Loaded Configuration File** line near the top of the output tells you exactly which `php.ini` PHP is reading.

For the full list of configuration directives, see [php.net's php.ini documentation](https://www.php.net/manual/en/configuration.file.php).

{: .tip }
> When `yum` updates PHP later, it preserves any config files you've modified and writes the new defaults alongside as `php.ini.rpmnew`. If you've made changes, diff against `.rpmnew` after every update so you don't miss new directives.

---

## Running multiple PHP versions side by side

A single IBM i partition can host as many PHP versions as you want, each fully isolated, using `chroot`. This is invaluable when you have one application stuck on PHP 7.4 and another on PHP 8.4, or when you need to test an upgrade in parallel without touching production.

`chroot` ("change root") makes a subdirectory look and behave like the root filesystem to processes running inside it. With it, you can install a complete second copy of `yum` plus its packages — including PHP — into an isolated tree.

### Install ibmichroot

```bash
yum install ibmichroot
```

### Create a chroot environment

The convention is one chroot per PHP version. Below we create one named `PHP73`, but you can name yours whatever fits your shop's conventions:

```bash
chroot_setup -y /QOpenSys/chroots/PHP73
```

This sets up a working `/QOpenSys/pkgs/` inside the chroot, with its own `yum`, its own configuration, and its own PHP install path.

To use the chroot, prefix commands with `chroot`:

```bash
chroot /QOpenSys/chroots/PHP73 /QOpenSys/pkgs/bin/php -v
```

You can run a different Apache instance inside the chroot, or have your top-level Apache forward FastCGI requests to a `php-cgi` running inside it. Either pattern works; choose based on which feels less surprising to operate.

{: .note }
> The chroot pattern is also useful for running NGINX and PHP-FPM together as a self-contained unit, separate from any Apache/PHP instance also running on the partition.

---

## Re-adding the IBM i base repos

If `yum-config-manager` ever loses track of the IBM repos (this happens occasionally after upgrades), re-add them with:

```bash
yum-config-manager --add-repo http://public.dhe.ibm.com/software/ibmi/products/pase/rpms/repo
```

This requires `yum-utils` to be installed already. If `yum` itself is broken, follow the [OSPM recovery guide](https://ospm.k3s.com) to bootstrap it.

---

## Where next

With PHP installed, you have two web-server choices ahead of you. Pick the one that fits your shop:

- **[Apache + PHP](apache.html)** — IBM's traditional path. Slightly more setup, well-documented, and what most existing IBM i shops are running. Configurable through the IBM HTTP Admin web interface.
- **[NGINX + PHP-FPM](nginx.html)** — the modern alternative. Hand-edited config, noticeably better performance under concurrent load, and the standard for new PHP deployments outside IBM i.
