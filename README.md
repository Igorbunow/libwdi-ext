# libwdi-ext — Extended libwdi fork for professional driver deployment

![VS2022](https://img.shields.io/badge/VS2022-passing-brightgreen.svg)
![MinGW](https://img.shields.io/badge/MinGW-passing-brightgreen.svg)
![Coverity](https://img.shields.io/badge/Coverity-passing-brightgreen.svg)
![Downloads](https://img.shields.io/github/downloads/Igorbunow/libwdi-ext/total.svg?color=brightgreen)
![License](https://img.shields.io/badge/License-GPLv3-blue.svg)

**libwdi-ext** is a feature-enhanced fork of  
[pbatard/libwdi](https://github.com/pbatard/libwdi)  
designed for embedded systems, OEM tools, automated provisioning, and advanced driver workflow scenarios.

The fork remains API-compatible with upstream while extending behavior where Windows driver installation is traditionally problematic.

---

## ⭐ Highlights

### ✔ External INF installation (legal-safe)
Install official FTDI/ST/other vendor drivers **without bundling them**:

- Direct `wdi_install_driver()`
- No need for `--with-userdir`
- Supports full driver directories (INF+SYS+CAT)

### ✔ OEM driver-store cleanup (no more `oemXX.inf` pollution)
Prevents Windows accumulating hundreds of stale driver entries.

Controlled via install options:

```c
struct wdi_options_install_driver id_options;
memset(&id_options, 0, sizeof(id_options));

// Keep the driver store clean by removing previous OEM INFs.
// By default, cleanup is enabled (disable_oem_inf_cleanup == 0).
// Set disable_oem_inf_cleanup = 1 to skip removal if needed.
id_options.disable_oem_inf_cleanup = 0; /* default: perform OEM INF cleanup */
```

### ✔ Fast installer mode (no 30–60s delay on Win10/11)

```c
id_options.no_syslog_wait = 1;
id_options.post_install_verify_timeout = 3000..5000;
```

### ✔ Extended Zadig features

* External INF selection & auto-detection
* Recursive INF search
* Status bar INF indication
* VID/PID/MI/driver-name filtering
* Driver visibility toggles
* Clean, deterministic installation cycle

### ✔ Fully backwards compatible

All original libwdi APIs continue to work.

---

## 📘 Documentation

Full embedding guide:

➡ **[docs/EMBEDDING_LIBWDI_EXTERNAL_INF.md](docs/EMBEDDING_LIBWDI_EXTERNAL_INF.md)**

➡ **[docs/Zadig_ext_ini_opts.md](docs/Zadig_ext_ini_opts.md)**

Includes examples for:

* WinUSB
* libusbK
* libusb-win32
* CDC
* USER(builtin)
* USER(external)

---

## 🚀 Getting Started

Build same as upstream:

```
./configure --with-wdf ...
make
```

`--with-userdir` is optional.

---

## 📄 Changelog

* **[CHANGELOG.md](CHANGELOG.md)** – libwdi extensions
* **[CHANGELOG_ZADIG.md](CHANGELOG_ZADIG.md)** – GUI improvements

---

## ❤️ Acknowledgements

Based on the outstanding upstream work of Pete Batard:

[https://github.com/pbatard/libwdi](https://github.com/pbatard/libwdi)

---

## 📜 License

This project remains under **GPLv3**, inherited from upstream libwdi.

Redistribution of *third-party proprietary drivers* is not allowed —
use **External INF mode** instead.

---

## 🤝 Contributing

See **CONTRIBUTING.md**

---

## 🔐 Security Policy

See **SECURITY.md**
