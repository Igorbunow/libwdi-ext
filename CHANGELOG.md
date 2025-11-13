# libwdi-ext Changelog

## [1.0.0-ext] — Extended Fork Initial Release
### Added
- External INF installation mode for `WDI_USER`
  - Allows installation of vendor-supplied drivers without redistributing them.
  - Supports direct `wdi_install_driver(dev, path, inf_name, ...)`.

- OEM INF cleanup system:
  - Controlled via `WDI_CLEANUP_OEM_INF` environment variable.
  - Removes stale driver-store entries (`oemXX.inf`) matching the same OriginalInfName.
  - Fixes long-standing Windows issue of duplicated driver records.

- Fast-exit installer options:
  - `id_options.no_syslog_wait`
  - `id_options.post_install_verify_timeout`
  - Eliminates 30–60 second delays on Windows 10/11.

### Improved
- More predictable driver re-enumeration timing.
- Better error reporting for missing external INF.
- Stability on Windows 10/11
- Clean installer teardown
- Error reporting for missing external INF

### Documentation
- Added `docs/EMBEDDING_LIBWDI_EXTERNAL_INF.md` with complete integration examples.

### Compatibility
- 100% API compatible with libwdi 1.5.x.
