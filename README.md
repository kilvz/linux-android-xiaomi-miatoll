# Kali NetHunter Kernel — Xiaomi MIATOLL (joyeuse)

Fork of [droidian-devices/linux-android-xiaomi-miatoll](https://github.com/droidian-devices/linux-android-xiaomi-miatoll) with full Kali NetHunter support and qcacld-3.0 built-in monitor mode fixes.

**Device**: Xiaomi MIATOLL (joyeuse) — SM7125 (Snapdragon 720G)  
**Kernel**: 4.14.336  
**Defconfig**: `gnumdk_defconfig`  
**Platform**: Droidian (Debian trixie)

---

## What's Added

### NetHunter patches (12 patches applied)
- RTL8812AU (88XXAU) — 5GHz AC WiFi with monitor mode + injection
- RTW88 — RTL8822BU/RTL8822CU USB WiFi
- RTL8187 — classic injection dongle support
- RTL8192CU — Realtek USB WiFi
- QCA frame injection — built-in QCA WiFi injection support
- Regulatory domain relaxation

### qcacld-3.0 Built-in Monitor Mode Fixes
The Qualcomm WCN3660 (built-in WLAN) now supports stable monitor mode via **both** the `change_iface` and `con_mode` paths.

#### Fix 1: NULL deref in injection workqueue (`ol_txrx.c`)
Crash: `ol_txrx_get_mon_vdev_from_pdev+0x14/0x24` — NULL dereference of `pdev->monitor_vdev` called from `hdd_process_injection_queue_work` after `hdd_vdev_destroy`.
Fix: Added NULL check on `pdev->monitor_vdev` and flush the injection workqueue before vdev destroy in `change_iface`.

#### Fix 2: PE session table cleanup (`wlan_hdd_main.c`)
Crash: stale PE session (`gpSession[vdev_id]` with `valid=true` but dangling pointers) causes panic on subsequent `ifup`.
Fix: Call `sme_delete_mon_session()` before `hdd_vdev_destroy()` in `hdd_stop_adapter()` for `QDF_MONITOR_MODE`.

#### Fix 3: ifdown ifup no-op (`wlan_hdd_main.c`)
Crash: `__hdd_stop()` called from `ndo_stop` triggered dangerous queue control operations in monitor mode.
Fix: Simplified `__hdd_stop()` to just set `DEVICE_IFACE_OPENED` and return 0 for monitor mode.

#### Fix 4: vdev destroy ordering (`wlan_hdd_main.c`)
Fix: Added `hdd_vdev_destroy()` after `hdd_deinit_adapter()` in the `QDF_MONITOR_MODE` block of change_iface.

### Kernel config highlights

| Category | Features |
|---|---|
| **USB Gadget** | HID keyboard/mouse, RNDIS, ECM, NCM, serial, ACM, OBEX, UVC (webcam), UAC1/UAC2 (audio), printer |
| **WiFi dongles** | ATH9K_HTC (AR9271), CARL9170, AR5523, RT2800USB, RT73USB, ZD1211RW, MT7601U, RTL8XXXU, P54_USB |
| **Bluetooth** | BT_HCIBTUSB (all USB dongles), BT_HCIUART, BT_DEBUGFS, BT_6LOWPAN |
| **Packet manipulation** | TC actions: pedit, nat, csum, ipt, vlan, bpf, connmark, skbmod, flower classifier, pktgen |
| **Tunneling** | IPIP, GRE, GENEVE, VXLAN, IPsec (AH/ESP) |
| **USB serial** | Option (GSM/CDMA), Simple, Garmin, Navman, Qualcomm, Sierra Wireless, debug |
| **USB/IP** | USB over IP (VHCI, host, VUDC) |
| **Netfilter** | SIP/SNMP conntrack, SYNPROXY, ECN, timeout policies, physdev/connlabel/IPVS matching, ARP/netdev logging |
| **IPVS** | Full IP Virtual Server with all schedulers and FTP helper |
| **SocketCAN** | CAN bus subsystem (automotive/OT security tools) |
| **Performance** | BPF JIT (ARM64), BBR + Cubic TCP, BFQ + Kyber + Deadline I/O, SCHED_AUTOGROUP, PSI, kexec, HZ=250 |
| **Filesystems** | NTFS, SquashFS LZ4/ZSTD, NFS server |
| **Networking** | Ingress qdisc, packet diagnostics, mac80211 debugfs, Broadcom BT UART |

### WireGuard
Already present and built as a module (`CONFIG_WIREGUARD=m`).

---

## Monitor Mode Usage

### Method A: `con_mode` sysfs (preferred)
Performs a complete firmware idle shutdown/restart cycle — most reliable.

```bash
# Monitor mode
svc wifi disable
ip link set wlan0 down
echo 4 > /sys/module/wlan/parameters/con_mode
ip link set wlan0 up

# Managed mode (STA)
ip link set wlan0 down
echo 0 > /sys/module/wlan/parameters/con_mode
ip link set wlan0 up
svc wifi enable
```

Valid `con_mode` values:
- `0` — QDF_GLOBAL_MISSION_MODE (managed STA)
- `4` — QDF_GLOBAL_MONITOR_MODE (monitor)

### Method B: `change_iface` (`iw`)
Simple but may have edge-cases with interface-down transitions. Channel setting via `iw` may not work; the card stays on the last channel from managed mode.

```bash
iw dev wlan0 set type monitor
ip link set wlan0 up

# Switch back
iw dev wlan0 set type managed
ip link set wlan0 up
```

### Channel Setting
With `con_mode` (method A), channel can be set via `iw`:
```bash
iw dev wlan0 set freq 5200   # 5GHz channel 40
iw dev wlan0 set channel 3   # 2.4GHz
```
With `change_iface` (method B), `iw set freq/channel` may return `Invalid argument` — the card remains on the last channel from managed association.

### Frame Injection Test
```bash
# Directed deauth (replace MACs)
aireplay-ng -0 2 -a <AP_BSSID> -c <CLIENT_MAC> --ignore-negative-one -D wlan0

# Broadcast deauth
aireplay-ng -0 2 -a <AP_BSSID> --ignore-negative-one -D wlan0
```

---

## How to Verify the Kernel Fixes

1. **Mode switch stability** — cycle monitor/managed 5+ times:
   ```bash
   for i in 1 2 3 4 5; do
     iw dev wlan0 set type monitor && sleep 1
     iw dev wlan0 set type managed && sleep 1
   done
   ```

2. **ifdown/ifup** — verify no crash with interface DOWN before mode switch:
   ```bash
   ip link set wlan0 down
   iw dev wlan0 set type monitor
   ip link set wlan0 up
   # if this doesn't crash, the fixes are working
   ```

3. **Injection** — deauth a connected client:
   ```bash
   aireplay-ng -0 1 -a <BSSID> -c <CLIENT_MAC> --ignore-negative-one -D wlan0
   ```

4. **Check kernel version** — verify your build is running:
   ```bash
   cat /proc/version
   uname -r
   ```

---

## Build

Build is done via `releng-build-package` inside a Docker container:

```bash
docker run --rm -v $PACKAGES_DIR:/buildd -v $KERNEL_DIR:/buildd/sources \
  -it quay.io/droidian/build-essential:trixie-amd64 bash

# Inside container:
rm -rf /buildd/sources/out/
rm -f debian/control
debian/rules debian/control
RELENG_HOST_ARCH="arm64" releng-build-package
```

### Toolchain
Uses `clang-android-6.0-4691093` as specified in `kernel-info.mk`.  
Required packages (install before building):
- `clang-android-6.0-4691093`
- `binutils-aarch64-linux-gnu`
- `gcc-4.9-aarch64-linux-android`
- `g++-4.9-aarch64-linux-android`
- `libgcc-4.9-dev-aarch64-linux-android-cross`
- `linux-initramfs-halium-generic:arm64`
- `python3` + `python-is-python3`
- `device-tree-compiler`
- `cpio`

### Docker Build
A Dockerfile is available at `docker/Dockerfile`. Build with:
```bash
docker build -t droidian-kernel-builder docker/
docker run --rm -v /tmp/opencode/build-out:/buildout droidian-kernel-builder \
  bash -c "cp /buildd/*.deb /buildout/ 2>/dev/null; ls -la /buildout/"
```

---

## Installation

Copy the `.deb` packages to the phone (via ADB) and install:

```bash
# Install in order:
dpkg -i linux-headers-4.14-336-xiaomi-miatoll_*.deb
dpkg -i linux-image-4.14-336-xiaomi-miatoll_*.deb
dpkg -i linux-bootimage-4.14-336-xiaomi-miatoll_*.deb
dpkg -i linux-libc-dev_*.deb
```

The `bootimage` postinst automatically flashes the boot partition.  
**Note**: Both `boot` (sde51) and `bootbak` (sde55) must be flashed.

Reboot to load the new kernel. Verify with `cat /proc/version`.

---

## Known Issues

- `change_iface` with `iff_up=false` (interface DOWN before conversion) may still have edge-cases. Use `con_mode` method for reliable transitions.
- ACK reporting in `aireplay-ng` shows `0| 0 ACKs` under `change_iface` path — packets are still transmitted and received by the AP.
- `iw dev wlan0 set freq/channel` only works under `con_mode` monitor, not `change_iface`.

---

## Files Changed

| File | Change |
|---|---|
| `drivers/staging/qcacld-3.0/core/hdd/src/wlan_hdd_main.c` | PE session cleanup, ifdown no-op, vdev destroy ordering |
| `drivers/staging/qcacld-3.0/core/dp/txrx/ol_txrx.c` | NULL check on `pdev->monitor_vdev` |
| `drivers/staging/qcacld-3.0/core/hdd/src/wlan_hdd_cfg80211.c` | Flush injection workqueue before vdev destroy |
| `debian/rules` | Build overrides, bootimage init_boot DTB fix |

---

## Credits

- [Droidian](https://droidian.org) — base kernel packaging
- [Kali NetHunter](https://www.kali.org/docs/nethunter/) — patches and injection framework
- [linux-packaging-snippets](https://github.com/droidian/linux-packaging-snippets) — build system
