# libwdi-ext — Extended libwdi fork for professional driver deployment

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

Enabled via:

```c
SetEnvironmentVariableA("WDI_CLEANUP_OEM_INF", "1");
````

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
