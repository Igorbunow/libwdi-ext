# Release Notes — libwdi-ext v1.0.0

## Summary
This is the first official release of **libwdi-ext**, an enhanced fork of
libwdi focused on clean driver installation, predictable behavior, and support
for external vendor INF packages.

## Most important features
- External INF installation mode
- OEM INF cleanup mechanism
- Faster driver installation (Win10/11)
- Extended Zadig features and filters

## Tested On
- Windows 7 SP1 x64
- Windows 10 Pro
- Windows 11 IoT Enterprise S
- FTDI FT2232H, custom CDC devices, libusb devices

## Known Limitations
- External INF must exist in a directory with the complete driver package.
- Driver signing requirements still apply for Win10+.
