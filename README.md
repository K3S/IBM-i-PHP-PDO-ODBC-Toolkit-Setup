# IBM i PHP / PDO / ODBC Toolkit Setup

A practical guide to running PHP on IBM i, connecting PHP applications to DB2, and calling RPG from PHP via the IBM i Toolkit.

**Read it at: [odbcphp.k3s.com](https://odbcphp.k3s.com)**

---

## What's here

This repository is the source for the guide site. The chapters cover:

- **Installing PHP on IBM i** — Zend RPMs, multiple versions via `chroot`.
- **Apache + PHP** with FastCGI, the IBM-traditional path.
- **NGINX + PHP-FPM**, the modern alternative.
- **Connecting PHP to DB2** via PDO and ODBC, both from IBM i and from Linux/Windows.
- **Calling RPG from PHP** via the Toolkit, reusing your existing PDO connection.
- **Running ODBC and Apache in custom subsystems** for isolation.
- **Performance tuning** — prestart jobs, exit programs, traces, FPM sizing.
- **Reference** — connection string keywords, repos, acknowledgments.

The site is a Jekyll project using the [just-the-docs](https://just-the-docs.github.io/just-the-docs/) theme, served from GitHub Pages with a custom domain.

---

## Companion guides

This is one of three K3S-published guides on IBM i development. The others are:

- **[RPG Tutorial](https://rpgtutorial.k3s.com)** — practical RPG programming through VS Code.
- **[IBM i AI Workers](https://ibmi-ai-workers.k3s.com)** — calling LLMs at scale from RPG.

Each is independent. Together they describe the K3S architecture: RPG owns business logic; PHP and a modern web framework own presentation and queries; AI is integrated as a worker pattern.

---

## Contributing

Issues and pull requests welcome. Use the **Edit this page on GitHub** link at the bottom of any page on the live site to jump straight to the source for a chapter.

---

## License

- **Prose** — [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/). See `LICENSE-PROSE`.
- **Code samples** — [MIT License](https://opensource.org/licenses/MIT). See `LICENSE`.

---

*Maintained by [King III Solutions Inc.](https://k3s.com)*
