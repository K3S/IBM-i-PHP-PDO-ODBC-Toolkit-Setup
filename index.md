---
title: Overview
layout: home
nav_order: 1
description: "A practical guide to running PHP on IBM i and connecting PHP applications to DB2 — including calling RPG from PHP via the Toolkit."
permalink: /
---

# PHP, PDO &amp; ODBC on IBM i

A practical guide for shops that want to run PHP on IBM i, connect PHP applications to DB2 from any platform, or call RPG programs from PHP using the IBM i Toolkit over ODBC.

This guide consolidates information that's scattered across IBM documentation, vendor sites, community blogs, and a decade of mailing-list threads — into one place, in a consistent voice, with examples that actually work.


[Get started](#start-here){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[View on GitHub](https://github.com/K3S/IBM-i-PHP-PDO-ODBC-Toolkit-Setup){: .btn .fs-5 .mb-4 .mb-md-0 }


---

## Who this is for

You'll get the most out of this if:

- You run, or are planning to run, PHP on an IBM i partition — under Apache or NGINX.
- You have a PHP application on Linux or Windows and need to query DB2 on IBM i over ODBC.
- You want to call RPG or CL programs from PHP, regardless of where PHP is running.
- You're separating concerns the way modern IBM i shops do: RPG owns business logic and writes; PHP and a modern web framework own presentation and queries.

You don't need to be an IBM i expert. We'll explain the things that are unusual *in this context* — file systems, libraries, subsystems, the Toolkit — and assume you already know what you'd expect a working PHP developer to know.

---

## Start here

- **Installing PHP on IBM i** via Zend's RPMs — the path most shops should take.
- **Multiple PHP versions side by side** using `chroot`, so you can run 7.4, 8.0, 8.3, and 8.4 on the same partition without conflict.
- **Apache + PHP** with FastCGI — IBM's traditional path, with the configuration that actually works.
- **NGINX + PHP-FPM** — the modern alternative, which performs noticeably better under load.
- **PDO + ODBC connections to DB2** — the connection string keywords that matter, the `php.ini` setting that turns slow queries fast, and examples for both PHP-on-IBM-i and PHP-on-Linux/Windows.
- **Calling RPG from PHP via the Toolkit** — including the trick of passing your existing PDO connection through to the Toolkit so you don't open a second one.
- **Operations** — running ODBC and Apache in their own subsystems for isolation, and a checklist of speed knobs that materially change behavior under load.
- **A reference page** — connection string keywords, the IBM repos, and acknowledgments to the people whose work this builds on.

---

## What this guide doesn't cover

- **Writing RPG.** If you're new to RPG itself, see our companion [RPG Tutorial](https://rpgtutorial.k3s.com).
- **Calling AI services from RPG at scale.** That's a separate problem with its own pattern — see [IBM i AI Workers](https://ibmi-ai-workers.k3s.com).
- **PHP language fundamentals.** We assume you can write a class, call a function, and read a stack trace.
- **Comparing frameworks.** We mention the frameworks K3S uses (currently Mezzio for our R7 platform; previously Laminas) where it's relevant context, but we don't argue for a particular choice.

---

## A note on the bigger picture

IBM i on Power Systems is one of the most stable, well-engineered enterprise platforms in production anywhere. The persistent perception that it's a dated platform comes mostly from two places: the green-screen 5250 interface that users associate with it, and the fact that its language history is unusually long (RPG, CL, COBOL, all still first-class). Neither of those things has anything to do with whether the platform is suitable for modern work.

K3S has run our customer-facing application on IBM i for thirty-plus years. We've modernized our stack steadily — most recently moving from Laminas-MVC to Mezzio (PSR-7 / PSR-15 middleware) for our R7 platform, with a React frontend on top — while keeping RPG as the system of record for business logic. The pattern below describes how that works.

We follow a [CQRS](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation) split: every database write goes through a single-purpose RPG API (an `ADD…`, `UPD…`, or `DLT…` program) so the business logic lives in one place and stays auditable. Reads come back through PHP over PDO/ODBC with whatever framework we're using at the time. The PHP layer is where the modern velocity lives; the RPG layer is where the durable logic lives. Either side can be replaced without breaking the other.

This guide is the practical foundation for setting up that pattern in your own shop.

---

## How to read this guide

The chapters are independent enough that you can jump straight to what you need. Use the sidebar to navigate, the search box at the top, and the **Edit this page on GitHub** link at the bottom of any page to suggest improvements.

If something is wrong, unclear, or missing context, please open an issue. We'd rather hear about it than not.

---

*Maintained by the team at [King III Solutions Inc.](https://k3s.com). Published at [odbcphp.k3s.com](https://odbcphp.k3s.com).*
