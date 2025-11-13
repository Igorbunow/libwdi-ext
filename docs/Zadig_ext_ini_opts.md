# Zadig-ext INI Options (extended)

This document describes the **Zadig-ext–specific** INI options that do **not**
exist in the original Zadig, or whose behavior has been extended.

All sections are optional.  
If a section or key is missing, Zadig-ext falls back to the original behavior.

The default INI file is still called `zadig.ini`.

---

## 1. `[updates]` — controlling online update checks

These options control Zadig-ext’s self-update check.  
They are **additive** to the original behavior.

```ini
[updates]
  # 1 — completely disable update checks
  disable = 1

  # (optional) override the base URL used for update checks
  # base_url = https://my-company.example.com/zadig/
````

### `disable`

* **Type:** integer / boolean (`0` or `1`)
* **Default:** `0` (updates enabled)
* `1` – Zadig-ext will not perform any online update checks.
* Useful in:

  * Offline / air-gapped environments
  * Corporate deployments where update endpoints are controlled centrally

### `base_url`

* **Type:** string (URL)
* **Default:** compiled-in upstream URL
* Overrides the base URL used to query for:

  * New Zadig-ext versions
  * Optional metadata
* Intended for:

  * OEM forks
  * Private mirrors / internal update servers

If `disable = 1`, `base_url` is ignored.

---

## 2. `[external_inf]` — external USER driver mode

These options enable the **external INF** mechanism for `WDI_USER` drivers.
This is a major extension compared to the original Zadig:

* You can reuse vendor-supplied drivers **without bundling them**.
* Zadig-ext installs the driver directly from a directory on disk.

```ini
[external_inf]
  # Enable external INF usage mode
  enabled = true

  # Path to the directory containing ftdibus.inf from the official package
  path    = C:\Drivers\FTDI_CDM\
```

### `enabled`

* **Type:** boolean (`true`/`false`, `1`/`0`)
* **Default:** `false`
* When `true` **and** `path` is not empty:

  * Zadig-ext allows the **USER** driver entry to appear in the driver list
    even if libwdi was built **without** `--with-userdir`.
  * When the user selects the **USER** driver, Zadig-ext:

    * Skips `wdi_prepare_driver()` for USER
    * Calls `wdi_install_driver()` directly with:

      * `path` = `[external_inf].path`
      * `inf`  = `[driver].user_inf`

### `path`

* **Type:** string (filesystem path)

* **Default:** empty

* Must point to a directory that contains:

  * `user_inf` file (see `[driver].user_inf`)
  * All other driver files required by the vendor:

    * `.sys`, `.cat`, DLLs, etc.

* Example for FTDI:

  ```ini
  [driver]
    user_inf = ftdibus.inf

  [external_inf]
    enabled = true
    path    = C:\Drivers\FTDI_CDM\
  ```

* Behavior:

  1. Zadig-ext first checks `path\user_inf`.
  2. If not found, Zadig-ext searches **recursively** under `path`.
  3. If still not found, Zadig-ext shows a **file selection dialog** to pick
     the INF manually.
  4. After a file is selected, Zadig-ext:

     * Updates `user_inf` and `path` internally
     * Shows an informational message suggesting you update the INI file
       so the configuration persists across runs.

---

## 3. `[behavior]` — installer behavior and driver visibility

These options control **how** Zadig-ext and the libwdi installer behave,
especially on modern Windows versions (10/11), and how drivers are shown in
the driver selection combo box.

```ini
[behavior]
  # instantly terminate, do not wait for tail setupapi.dev.log
  no_syslog_wait = true
  # or synonym:
  # disable_syslog_tail = 1

  # optional: quick check of actual service binding (ms)
  post_install_verify_timeout_ms = 2000

  # control OEM INF cleanup in the installer (default = true)
  cleanup_oem_inf = true

  # Driver visibility control
  show_winusb   = true
  show_cdc      = false
  show_libusb0  = false
  show_libusbk  = false
  show_user     = true
```

### 3.1 Installer timing and system log behavior

#### `no_syslog_wait` / `disable_syslog_tail`

* **Type:** boolean
* **Default:** `false`
* When enabled, Zadig-ext sets `id_options.no_syslog_wait = 1`, which:

  * disables waiting for the Windows event log / driver log tail
  * avoids long (30–60 second) pauses after installation on Windows 10/11
* `disable_syslog_tail` is a **synonym** for `no_syslog_wait`.

You should normally set:

```ini
no_syslog_wait = true
```

for interactive tools where user responsiveness matters.

#### `post_install_verify_timeout_ms`

* **Type:** integer (milliseconds)
* **Default:** fork-specific (typically `3000–5000`)
* Controls `id_options.post_install_verify_timeout`:

  * upper bound on how long Zadig-ext waits for the device to re-enumerate
    after driver installation
* Lower values:

  * Faster return if enumeration is slow or blocked
* Higher values:

  * More tolerance for slow hardware or hubs

Example:

```ini
post_install_verify_timeout_ms = 2000
```

gives a 2-second verification window.

### 3.2 OEM INF cleanup

#### `cleanup_oem_inf`

* **Type:** boolean

* **Default:** `true` (in this fork)

* Controls whether Zadig-ext tells the helper installer to:

  * Search the Windows driver store for existing `oemXX.inf` entries
  * Remove entries whose **OriginalInfName** matches the INF being installed

* Implemented via environment variable:

  ```c
  SetEnvironmentVariableA("WDI_CLEANUP_OEM_INF", "1" or "0");
  ```

* When `cleanup_oem_inf = true`:

  * repeated driver installations do **not** accumulate many duplicates
  * the driver store stays clean, even with frequent switching

You can disable this (for debugging only) with:

```ini
cleanup_oem_inf = false
```

### 3.3 Driver visibility options

These flags **only affect the GUI** (Zadig-ext), not the underlying libwdi API.
They determine which drivers are offered in the driver combo box.

#### `show_winusb`

* **Type:** boolean
* **Default:** `true`
* Shows or hides the **WinUSB** driver entry.

#### `show_cdc`

* **Type:** boolean
* **Default:** `false` (in your current configuration)
* Shows or hides the **USB Serial (CDC)** driver entry.

#### `show_libusb0`

* **Type:** boolean
* **Default:** `false` in your examples
* Controls visibility of the **libusb-win32** driver entry.

#### `show_libusbk`

* **Type:** boolean
* **Default:** `false` in your examples
* Controls visibility of the **libusbK** driver entry.

#### `show_user`

* **Type:** boolean
* **Default:** `true`
* Controls visibility of the **USER** driver entry:

  * Built-in USER (from `--with-userdir`) and/or
  * External USER (when `[external_inf].enabled = true`)

If any of these flags is `false`, that driver:

* is not shown in the list,
* is not chosen as default driver type.

---

## 4. `[driver]` — USER driver configuration and extraction behavior

These options control the special **USER** driver entry, both for:

* Built-in USER dir (compiled via `--with-userdir`)
* External INF mode (paired with `[external_inf]`)

```ini
[driver]
  # INF name from the built-in userdir, or external vendor INF
  # used when "Custom/WDI_USER" is selected
  user_inf = ftdibus.inf

  # Extract driver files only, don't install (default = false)
  extract_only = false

  # Display name template for the USER driver entry
  user_display = "FTDI CDM ({inf})"
```

### `user_inf`

* **Type:** string (filename)
* **Default:** implementation-defined; in your config: `ftdibus.inf`
* Used in two scenarios:

1. **Built-in USER driver (with `--with-userdir`)**

   * `user_inf` is the INF file inside the userdir package.
   * Zadig-ext calls `wdi_prepare_driver()` + `wdi_install_driver()` using this INF.

2. **External INF mode**

   * `user_inf` is the **vendor INF filename** in `[external_inf].path`.
   * Zadig-ext:

     * Validates its existence (`path\user_inf`)
     * Installs directly from that directory via `wdi_install_driver()`.

### `extract_only`

* **Type:** boolean
* **Default:** `false`
* When `true`, Zadig-ext:

  * Only calls `wdi_prepare_driver()` to extract files into the target directory
  * **Does not** call `wdi_install_driver()`
* Useful for:

  * Debugging generated packages
  * Custom deployment tooling where another process performs the installation

When `extract_only = false` (normal mode), Zadig-ext performs both extraction and installation.

### `user_display`

* **Type:** string (template)
* **Default:** `"Custom driver ({inf})"` or similar; in your config: `"FTDI CDM ({inf})"`
* Controls the label shown in the driver combo box for the USER driver entry.
* The `{inf}` placeholder is replaced with the actual INF filename
  (`user_inf` value).

Example:

```ini
user_inf     = ftdibus.inf
user_display = "FTDI CDM ({inf})"
```

→ visible in UI as:

> `FTDI CDM (ftdibus.inf)`

When using external INF mode, this allows users to clearly recognize which
vendor driver they are installing.

---

## 5. `[device]` (extended filtering — summary)

Although not listed in your snippet, Zadig-ext also adds device filtering
options that were not present in the original Zadig.
They are briefly summarized here for completeness:

Typical usage:

```ini
[device]
  vid    = 0x0403
  pid    = 0x6010,0x6011
  vid_pid = 0x0403:0x6010,0x0403:0x6011
  mi     = 1
  driver = usbser,winusb
```

Implementation details may vary, but in general:

* **`vid`** — filter by one or more Vendor IDs
* **`pid`** — filter by one or more Product IDs
* **`vid_pid`** — filter by explicit VID:PID pairs
* **`mi`** — filter by interface number (MI); useful for composite devices (e.g. FT2232H interface 0 vs 1)
* **`driver`** — filter by currently bound driver (e.g. `"usbser"`, `"winusb"`, `"ftdibus"`)

If any of these options are present, Zadig-ext logs something like:

> `Device filter enabled: VIDs=1, PIDs=2, MI=1, VID/PID pairs=2, Drivers=2`

and only devices matching the filter are shown.

---

## 6. Putting it together — example INI

```ini
[updates]
  disable  = 1
  # base_url = https://my-company.example.com/zadig/

[external_inf]
  enabled = true
  path    = C:\Drivers\FTDI_CDM\

[behavior]
  no_syslog_wait                 = true
  post_install_verify_timeout_ms = 2000
  cleanup_oem_inf                = true

  show_winusb   = true
  show_cdc      = false
  show_libusb0  = false
  show_libusbk  = false
  show_user     = true

[driver]
  user_inf     = ftdibus.inf
  extract_only = false
  user_display = "FTDI CDM ({inf})"

[device]
  vid     = 0x0403
  pid     = 0x6010,0x6011
  mi      = 1
  driver  = usbser,winusb
```

This configuration:

* Disables online updates
* Uses the official FTDI package from `C:\Drivers\FTDI_CDM\`
* Keeps installations fast on Win10/11
* Cleans up stale OEM INF entries
* Shows only WinUSB + USER in the driver list
* Restricts the device list to FTDI FT2232H, interface 1, with specific drivers

This captures the extended behavior you have added to **Zadig-ext** compared
to the original Zadig.
