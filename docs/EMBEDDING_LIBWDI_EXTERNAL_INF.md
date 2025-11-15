# Embedding libwdi (enhanced fork) into your own application

This document shows how to use **libwdi** from C code to install USB drivers
from your own application, using this fork's extended behavior.

It covers:

- Installing standard libwdi-generated drivers:
  - **WinUSB**
  - **USB Serial (CDC)**
  - **libusb-win32**
  - **libusbK**
- Installing a **USER** driver from a built–in package (`--with-userdir`)
- Installing a **USER** driver from an **external vendor INF**
  (without bundling the driver into your binary)
- Using the extended install options to:
  - avoid long waits on Windows 10/11 caused by system log tailing
  - clean up duplicate OEM INF entries created by repeated driver installs

All examples are written in C and assume you have included:

```c
#include "libwdi.h"
````

and linked against this fork of `libwdi`.

---

## 1. Common setup: enumerating devices and helpers

A typical flow is:

1. Create a device list with `wdi_create_list()`
2. Pick a device (e.g. by VID/PID/interface)
3. Prepare driver files with `wdi_prepare_driver()` (for libwdi–generated drivers)
4. Install the driver with `wdi_install_driver()`

Below is a minimal helper that finds the first device matching a given VID/PID:

```c
static struct wdi_device_info* find_device(WORD vid, WORD pid)
{
    struct wdi_device_info *list = NULL, *dev;

    if (wdi_create_list(&list, NULL) != WDI_SUCCESS) {
        return NULL;
    }

    for (dev = list; dev != NULL; dev = dev->next) {
        if (dev->vid == vid && dev->pid == pid) {
            return dev; // NOTE: list is kept alive; free it after use.
        }
    }

    wdi_destroy_list(list);
    return NULL;
}
```

In a real application you will usually:

* pass `dev` (the selected `struct wdi_device_info`) to your install function
* only call `wdi_destroy_list(list)` **after** installation is done.

For simplicity, examples below assume that:

```c
struct wdi_device_info* dev = find_device(0x0403, 0x6010); // FTDI FT2232H, for example
```

and that `dev != NULL`.

---

## 2. Installing standard libwdi–generated drivers

(WinUSB, libusb-win32, libusbK, CDC)

For drivers that libwdi generates for you (WinUSB, libusb–based drivers, CDC),
you typically:

1. Configure `wdi_options_prepare_driver`
2. Call `wdi_prepare_driver()` to create the driver package in a temporary directory
3. Configure `wdi_options_install_driver`
4. Call `wdi_install_driver()` to actually install the driver

### 2.1 Shared install options (no_syslog_wait, post_install_verify_timeout)

This fork extends `wdi_options_install_driver` with useful fields:

```c
struct wdi_options_install_driver id_options;
memset(&id_options, 0, sizeof(id_options));

// hwnd for UI / message boxes (can be NULL for console apps)
id_options.hWnd = hwnd; // or NULL

// Avoid long waits for system log tailing on Windows 10/11
id_options.no_syslog_wait = 1;

// Shorten post-install verification timeout (milliseconds)
// This controls how long libwdi waits for the device to re-enumerate.
id_options.post_install_verify_timeout = 5000; // e.g. 5 seconds
```


```c
// Enable removal of previous OEM INF files with matching OriginalInfName
id_options.disable_oem_inf_cleanup = 0;
```

### 2.2 WinUSB example

```c
#include "libwdi.h"

int install_winusb_driver(struct wdi_device_info* dev)
{
    int r;
    struct wdi_options_prepare_driver pd_options;
    struct wdi_options_install_driver id_options;
    char inf_name[256] = {0};
    char target_dir[MAX_PATH] = "C:\\\\Temp\\\\libwdi_driver";

    if (dev == NULL) {
        return WDI_ERROR_INVALID_PARAM;
    }

    memset(&pd_options, 0, sizeof(pd_options));
    memset(&id_options, 0, sizeof(id_options));

    // 1) Prepare: WinUSB
    pd_options.driver_type     = WDI_WINUSB;
    pd_options.vendor_name     = "My Company";
    pd_options.device_guid     = "{CDB3B5AD-293B-4663-AA36-1AAE46463776}"; // example GUID
    pd_options.disable_cat     = 0;   // generate .cat
    pd_options.disable_signing = 0;   // allow cat signing if available
    pd_options.use_wcid_driver = 0;   // classic WinUSB INF, not WCID

    // Use a valid file name based on device description
    if (wdi_get_inf_name(dev, inf_name, sizeof(inf_name)) != WDI_SUCCESS) {
        return WDI_ERROR_INVALID_PARAM;
    }

    r = wdi_prepare_driver(dev, target_dir, inf_name, &pd_options);
    if (r != WDI_SUCCESS) {
        return r;
    }

    // 2) Install
    id_options.hWnd                         = NULL; // or your window handle
    id_options.no_syslog_wait              = 1;     // avoid long waits
    id_options.post_install_verify_timeout = 5000;  // 5s verification timeout

    // Enable OEM INF cleanup in the helper installer process
    id_options.disable_oem_inf_cleanup = 0;

    r = wdi_install_driver(dev, target_dir, inf_name, &id_options);
    return r;
}
```

### 2.3 libusb-win32 (libusb0), libusbK, CDC examples

The code is identical to WinUSB, you only change `driver_type`
(and possibly the GUID and vendor name):

```c
// libusb-win32:
pd_options.driver_type = WDI_LIBUSB0;

// libusbK:
pd_options.driver_type = WDI_LIBUSBK;

// USB CDC:
pd_options.driver_type = WDI_CDC;
```

Everything else (prepare + install + `id_options` tuning) remains the same.

---

## 3. Installing USER driver from a built-in package (USERDIR)

If you build libwdi with `--with-userdir=...` (USERDIR support), you can ship
a pre-built vendor driver package (INF + SYS + CAT + DLLs) compiled into your
application. In this case:

* you set `pd_options.driver_type = WDI_USER`
* `wdi_prepare_driver()` knows how to extract the built-in package into
  the target directory
* `wdi_install_driver()` installs that extracted package as usual

Example:

```c
int install_user_builtin_driver(struct wdi_device_info* dev)
{
    int r;
    struct wdi_options_prepare_driver pd_options;
    struct wdi_options_install_driver id_options;
    char inf_name[256] = "ftdibus.inf";          // example INF name inside USERDIR
    char target_dir[MAX_PATH] = "C:\\\\Temp\\\\my_user_driver";

    if (dev == NULL) {
        return WDI_ERROR_INVALID_PARAM;
    }

    memset(&pd_options, 0, sizeof(pd_options));
    memset(&id_options, 0, sizeof(id_options));

    // USER driver from built-in USERDIR
    pd_options.driver_type     = WDI_USER;
    pd_options.vendor_name     = "FTDI CDM driver (builtin)";
    pd_options.device_guid     = "{CDB3B5AD-293B-4663-AA36-1AAE46463776}";
    pd_options.disable_cat     = 0;
    pd_options.disable_signing = 0;
    pd_options.use_wcid_driver = 0;

    r = wdi_prepare_driver(dev, target_dir, inf_name, &pd_options);
    if (r != WDI_SUCCESS) {
        return r;
    }

    id_options.hWnd                         = NULL;
    id_options.no_syslog_wait              = 1;
    id_options.post_install_verify_timeout = 5000;

    id_options.disable_oem_inf_cleanup = 0;

    r = wdi_install_driver(dev, target_dir, inf_name, &id_options);
    return r;
}
```

This approach is useful when you are legally allowed to **redistribute**
the full driver package and want it embedded into your application.

---

## 4. Installing USER driver from an external vendor INF (this fork)

In some cases you **cannot** or do not want to redistribute third-party
drivers (FTDI, ST, etc.) inside your application. Instead, the user:

1. Downloads and installs or unpacks the vendor's official driver package
2. Points your application (or Zadig) to the directory that contains the INF

This fork supports an **external INF mode** for `WDI_USER` where:

* you skip `wdi_prepare_driver()`
* you call `wdi_install_driver()` directly with:

  * `path` = directory that contains the INF/SYS/CAT…
  * `inf_name` = name of the vendor INF file (e.g. `ftdibus.inf`)

### 4.1 Minimal example (external INF)

```c
int install_user_external_inf(struct wdi_device_info* dev,
                              const char* driver_dir,
                              const char* inf_name)
{
    int r;
    struct wdi_options_install_driver id_options;
    char full_inf[MAX_PATH];
    DWORD attr;

    if ((dev == NULL) || (driver_dir == NULL) || (inf_name == NULL)) {
        return WDI_ERROR_INVALID_PARAM;
    }

    // sanity check: INF must exist at driver_dir\\inf_name
    _snprintf(full_inf, sizeof(full_inf), "%s\\\\%s", driver_dir, inf_name);
    attr = GetFileAttributesA(full_inf);
    if (attr == INVALID_FILE_ATTRIBUTES || (attr & FILE_ATTRIBUTE_DIRECTORY)) {
        // caller can log or display an error here
        return WDI_ERROR_NOT_FOUND;
    }

    memset(&id_options, 0, sizeof(id_options));
    id_options.hWnd                         = NULL; // or your HWND
    id_options.no_syslog_wait              = 1;
    id_options.post_install_verify_timeout = 5000;

    // Enable OEM INF cleanup in the helper installer process
    id_options.disable_oem_inf_cleanup = 0;

    // NOTE: no wdi_prepare_driver() here – we install directly from the vendor package
    r = wdi_install_driver(dev, driver_dir, inf_name, &id_options);
    return r;
}
```

**Important notes:**

* `driver_dir` must contain the **full driver package**, not just the INF:

  * `ftdibus.inf`, `.sys`, `.cat`, etc.
* This mode is especially useful when:

  * The user has already installed or unpacked the official vendor drivers
  * You want to switch the binding to that driver using libwdi
  * You do **not** want to redistribute the driver yourself

This pattern mirrors what the enhanced Zadig build does when configured with
an `[external_inf]` section in `zadig.ini`.

---

## 5. Avoiding system pollution from repeated installs

(OEM INF cleanup)

By default, each driver installation through libwdi (or the Windows driver
installer) can create a new `oemXX.inf` entry in the system driver store,
especially when you repeatedly reinstall or switch drivers for the same
device/interface.

This fork introduces a helper installer behavior where, **before** installing
a new driver package:

* it enumerates existing OEM INF entries
* looks at their `OriginalInfName`
* removes those whose `OriginalInfName` matches the INF being installed

Control is done via ``id_options.disable_oem_inf_cleanup`` variable evaluated by the installer process:

```c
// Enable aggressive OEM INF cleanup for the current installation
id_options.disable_oem_inf_cleanup = 0;
```

If you want to leave the system store untouched (for debugging or testing),
you can set it to `"0"`:

```c
id_options.disable_oem_inf_cleanup = 1;
```

For GUI usage (Zadig), this is typically wired to an ini option
such as:

```ini
[behavior]
cleanup_oem_inf = true
```

---

## 6. Avoiding long post-install delays (system log tailing)

On modern Windows versions (Windows 10/11), the standard log–tailing and
post–install verification steps can cause **long, unexpected pauses**
(around 30–60 seconds) after the driver appears to have been installed.

This fork exposes two fields in `wdi_options_install_driver` that help:

```c
struct wdi_options_install_driver id_options;
memset(&id_options, 0, sizeof(id_options));

// ...
id_options.no_syslog_wait              = 1;     // do not wait for system log tail
id_options.post_install_verify_timeout = 5000;  // how long to wait (ms) for re-enumeration
```

Recommended settings for interactive tools:

* `no_syslog_wait = 1` – fast and predictable exit
* `post_install_verify_timeout = 3000–5000` ms – short but reasonable
  delay to allow the device to reappear under its new driver

These options are used consistently in all examples above and help keep
driver switching operations responsive even on slower or heavily loaded systems.

---

## 7. Summary

To embed this enhanced fork of libwdi in your own application:

* Use `wdi_prepare_driver()` + `wdi_install_driver()` for:

  * **WinUSB**, **CDC**, **libusb-win32**, **libusbK**
  * **built–in USER** drivers compiled via `--with-userdir`
* Use **direct `wdi_install_driver()` with a vendor INF** when:

  * The driver is provided by the vendor (FTDI, ST, etc.)
  * You do not want to redistribute the driver
  * The user has the driver package on disk
* Always consider:

  * `id_options.no_syslog_wait`
  * `id_options.post_install_verify_timeout`
  * `id_options.disable_oem_inf_cleanup`

Together, these options help keep installations fast, avoid system
pollution from repeated installs, and integrate cleanly with official
vendor driver packages.
