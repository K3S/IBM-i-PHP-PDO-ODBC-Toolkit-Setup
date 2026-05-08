---
title: Reference
nav_order: 9
---

# Reference

A short reference page: the most-used ODBC connection string keywords, the IBM repos, and acknowledgments.

---

## ODBC connection string keywords

The full canonical list lives in IBM's documentation: [IBM i ODBC Connection String Keywords](https://www.ibm.com/docs/en/i/7.5?topic=details-connection-string-keywords). The table below covers the keywords you'll use 95% of the time.

| Keyword | Values | What it does |
|---------|--------|--------------|
| `DRIVER` | `{IBM i Access ODBC Driver}` | The ODBC driver to use. Curly braces are required. |
| `SYSTEM` | hostname or IP | The IBM i partition to connect to. |
| `UID` | user profile | Authentication user. |
| `PWD` | password | Authentication password. |
| `NAM` | `0` (SQL) / `1` (System) | Naming convention. `1` = `*SYS`, slash-separated, library-list-aware. Use `1` for legacy IBM i applications. |
| `DBQ` | `default, lib1 lib2 ...` | Library list. Default library before the comma; additional libraries after, space-separated. Leading comma means "no default library." |
| `CCSID` | numeric | Character set for data on the wire. `1208` = UTF-8, the right choice for cross-platform. |
| `TSFT` | `0` / `1` | Timestamp format. `1` returns IBM i format (`YYYY-MM-DD-HH.MM.SS.uuuuuu`). |
| `DBGSRV` | `0` / `1` | Enable Db2 for i debug services for this connection. |
| `XMD` | `0` / `1` | Trim trailing blanks from CHAR data. `1` is usually what you want for PHP. |
| `CMT` | `0` / `1` / `2` / `3` | Commitment control level. `0` = `*NONE` (default), `1` = `*CHG`, `2` = `*CS`, `3` = `*ALL`. |
| `BLOCKFETCH` | `0` / `1` | Enable result set blocking. `1` is usually faster for large result sets. |
| `LAZYCLOSE` | `0` / `1` | Defer closing cursors until the connection closes. Faster but holds locks longer. |
| `EXTCOLINFO` | `0` / `1` | Return extended column metadata (column headings, descriptions). |

---

## IBM i open-source repositories

If `yum-config-manager` ever loses track of the IBM repos, re-add them:

```bash
yum-config-manager --add-repo http://public.dhe.ibm.com/software/ibmi/products/pase/rpms/repo
```

To add the Zend PHP repository:

```bash
yum-config-manager --add-repo http://repos.zend.com/ibmiphp/
```

For the full list of community-maintained third-party repositories: [3rd Party Open Source Repos for IBM i](https://ibmi-oss-docs.readthedocs.io/en/latest/yum/3RD_PARTY_REPOS.html).

If `yum` itself is broken and you need to bootstrap from scratch without SSH, see [How to set up OSPM without SSH](https://ospm.k3s.com).

---

## Companion guides

The K3S tutorial network covers adjacent topics:

- **[RPG Tutorial](https://rpgtutorial.k3s.com)** — practical RPG programming through VS Code, with progressive examples covering free-format RPGLE, DB2, service programs, error handling, and the XMLSERVICE bridge.
- **[IBM i AI Workers](https://ibmi-ai-workers.k3s.com)** — calling LLMs at scale from RPG batch jobs, using PHP as the transport layer and data queues as the coordination point. Multi-tenant, retries, observability.

---

## Acknowledgments

This guide consolidates information from many sources. Particular thanks to:

- **[Chuk Shirley](https://github.com/chukShirley)** — for years of patient explanations and shared examples.
- **[Stephanie Rabbini](https://twitter.com/jordiwes)** — Seiden Group, on PHP and IBM i.
- **[Alan Seiden](https://twitter.com/alanseiden)** — Seiden Group, whose published work is a foundational reference for the entire IBM i PHP community.
- **[Dave Dressler](https://godzillai5.wordpress.com/)** — for the [Godzilla i5](https://godzillai5.wordpress.com/) writeups.
- **[Dawn May](https://dawnmayi.com/)** — whose blog is the canonical reference on IBM i subsystems and routing.
- **[Kevin Adler](https://twitter.com/kadler_ibm)** — IBM, on PASE and the open-source ecosystem.
- **[Liam Allan](https://github.com/worksofliam)** — for Code for IBM i and the modern toolchain that makes all of this far easier than it used to be.
- **[Scott Klement](https://www.scottklement.com/)** — whose articles remain the best writing on modern RPG and IBM i integration anywhere.

The mistakes are ours. The good ideas are theirs.

---

## License

The prose of this guide is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/). Share, adapt, translate, build on it — including for commercial purposes — as long as you provide attribution to K3S and link back.

Code samples are licensed under the [MIT License](https://opensource.org/licenses/MIT). Use them freely.

See `LICENSE` and `LICENSE-PROSE` in the repository root for full text.

---

## Contributing, questions, corrections

If something is wrong, unclear, or missing context that would have helped you, please [open an issue](https://github.com/K3S/IBM-i-PHP-PDO-ODBC-Toolkit-Setup/issues) or use the **Edit this page on GitHub** link at the bottom of any page.

If you've solved a problem this guide doesn't cover and want to contribute a section, pull requests are welcome.
