# Kali NetHunter Kernel — Xiaomi MIATOLL (joyeuse)

Fork of [droidian-devices/linux-android-xiaomi-miatoll](https://github.com/droidian-devices/linux-android-xiaomi-miatoll) with full Kali NetHunter support.

**Device**: Xiaomi MIATOLL (joyeuse) — SM7125 (Snapdragon 720G)  
**Kernel**: 4.14.336  
**Defconfig**: `gnumdk_defconfig`

## What's added

### NetHunter patches (12 patches applied)
- RTL8812AU (88XXAU) — 5GHz AC WiFi with monitor mode + injection
- RTW88 — RTL8822BU/RTL8822CU USB WiFi
- RTL8187 — classic injection dongle support
- RTL8192CU — Realtek USB WiFi
- QCA frame injection — built-in QCA WiFi injection support
- Regulatory domain relaxation

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

## Build

Build is done via `releng-build-package` inside a Docker container:

```bash
docker run --rm -v $PACKAGES_DIR:/buildd -v $KERNEL_DIR:/buildd/sources \
  -it quay.io/droidian/build-essential:trixie-amd64 bash

# Inside container:
rm -f debian/control
debian/rules debian/control
RELENG_HOST_ARCH="arm64" releng-build-package
```

### Toolchain
Uses `clang-android-6.0-4691093` as specified in `kernel-info.mk`.

## Flashing
The build produces `.deb` packages in `/buildd/` including `linux-bootimage-*.deb` (contains `boot.img`, `recovery.img`, `dtbo.img`, `vbmeta.img`) and `linux-image-*.deb`.
