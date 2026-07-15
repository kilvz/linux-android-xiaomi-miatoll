# Device info

- codename: miatoll (joyeuse)
- soc: qualcomm sm7125 (snapdragon 720g)
- platform: atoll
- kernel: 4.14-336-xiaomi-miatoll
- os: Droidian (Debian trixie)

# Droidian Kernel Build

## TOOLCHAIN — MUST USE CLANG 6
- kernel-info.mk specifies `TOOLCHAIN = clang-android-6.0-4691093`
- Debian package name: `clang-android-6.0-4691093`
- Install path: `/usr/lib/llvm-android-6.0-4691093/bin/clang`
- Do NOT change to clang-10.0-r370808. User explicitly reverted back to clang 6.

## Build system
- Uses `linux-packaging-snippets` (repo: https://github.com/droidian/linux-packaging-snippets/tree/droidian) + `releng-build-package`
- `kernel-snippet.mk` reads `debian/kernel-info.mk`
- Build is run inside Docker container (quay.io/droidian/build-essential:trixie-amd64)
- Docker socket requires sudo (user not in docker group)
- DO NOT add declarations/EXPORT_SYMBOL to net/ethernet/eth.c or include/linux/etherdevice.h for driver-internal functions like is_broadcast_mac_addr — the 88XXau driver defines these itself via macros/extern __inline in its own ieee80211.h

## Droidian kernel build command
- Official build method (from https://docs.droidian.org/porting-guide/kernel-compilation/):
  ```
  (docker)# rm -f debian/control
  (docker)# debian/rules debian/control
  (docker)# RELENG_HOST_ARCH="arm64" releng-build-package
  ```
- The Docker container is started interactively:
  ```
  (host)$ docker run --rm -v $PACKAGES_DIR:/buildd -v $KERNEL_DIR:/buildd/sources -it quay.io/droidian/build-essential:trixie-amd64 bash
  ```
- Output packages go to `/buildd` (parent of sources), NOT `/buildd/out`.

## CRITICAL — fresh build from source
- **Always delete stale build artifacts** before building to prevent reuse of old cached kernel:
  ```
  (docker)# rm -rf /buildd/sources/out/
  ```
- The `out/KERNEL_OBJ/` directory may contain kernel built by CI with different hostname; `releng-build-package` will NOT rebuild from scratch if it finds existing build stamps.
- After `rm -rf out/`, proceed with the normal build commands.

## Build system overrides kernel version
- `kernel-snippet.mk` sets `KERNELRELEASE=$(KERNEL_BASE_VERSION)-$(DEVICE_VENDOR)-$(DEVICE_MODEL)` on the `make` command line.
- This overrides the kernel's own version calculation, so **`CONFIG_LOCALVERSION` from defconfig is NOT reflected in `uname -r`**.
- **Fix:** `debian/rules` overrides `KERNEL_BASE_VERSION` to `4.14-336-kilv` BEFORE the include, so the version becomes `4.14-336-kilv-xiaomi-miatoll`.
- `debian/rules` also sets `export KBUILD_BUILD_USER = kilv` so `/proc/version` shows `(kilv@...)` instead of `(root@<docker-id>)`.
- `uname -r` will show `4.14-336-kilv-xiaomi-miatoll` (includes `-kilv`).
- To verify the running kernel is your build: check `cat /proc/version` and look for the compile timestamp.

## Cross-toolchain requirements
- The `DEB_TOOLCHAIN` in kernel-info.mk lists ALL needed packages.
- When using `dpkg-buildpackage` directly, DO NOT use `-d` flag — it skips Build-Depends installation.
- Required packages (install in Dockerfile, not via dpkg-buildpackage -d):
  - `clang-android-6.0-4691093` (toolchain CC)
  - `binutils-aarch64-linux-gnu` (cross binutils)
  - `gcc-4.9-aarch64-linux-android` (provides `aarch64-linux-android-ld` + `aarch64-linux-android-gcc`)
  - `g++-4.9-aarch64-linux-android`
  - `libgcc-4.9-dev-aarch64-linux-android-cross`
  - `linux-initramfs-halium-generic:arm64`
  - `python3` + `python-is-python3` (kernel scripts need `/usr/bin/env python`)
  - `device-tree-compiler` (for DTB/DTBO compilation)
  - `cpio` (for initramfs creation)

## Package install on phone
- Install all .deb packages in order:
  ```
  (phone)# dpkg -i linux-headers-4.14-336-xiaomi-miatoll_*.deb
  (phone)# dpkg -i linux-image-4.14-336-xiaomi-miatoll_*.deb
  (phone)# dpkg -i linux-bootimage-4.14-336-xiaomi-miatoll_*.deb
  (phone)# dpkg -i linux-libc-dev_*.deb
  # plus any meta-packages
  ```
- The bootimage **postinst** automatically flashes the boot partition.
- Reboot to load the new kernel.
- Verify: `cat /proc/version` — check compile hostname matches your Docker container and timestamp matches build time.
- `uname -r` always shows `4.14-336-xiaomi-miatoll`; use `/proc/version` to distinguish builds.

## Copy debs from Docker to host
- After build, exit the container. The `.deb` files are in `/buildd/` (inside container).
- On the host, copy them from the container:
  ```
  (host)$ docker run --rm -v /tmp/opencode/build-out:/buildout droidian-kernel-builder bash -c "cp /buildd/*.deb /buildout/ 2>/dev/null; ls -la /buildout/"
  ```

## Docker build
- Dockerfile at /tmp/opencode/docker-build/Dockerfile
- Must bootstrap GPG keys (Droidian + Mobian repos have expired keys)
- Image name: droidian-kernel-builder
- Output dir (host): /tmp/opencode/build-out/

## Package install on phone
- Install all .deb packages in order:
  ```
  (phone)# dpkg -i linux-headers-4.14-336-kilv-xiaomi-miatoll_*.deb
  (phone)# dpkg -i linux-image-4.14-336-kilv-xiaomi-miatoll_*.deb
  (phone)# dpkg -i linux-bootimage-4.14-336-kilv-xiaomi-miatoll_*.deb
  # plus any meta-packages
  ```
- The bootimage **postinst** automatically flashes the boot partition.
- Reboot to load the new kernel.
- Verify: `cat /proc/version` — check compile hostname matches your Docker container and timestamp matches build time.
- `uname -r` shows `4.14-336-kilv-xiaomi-miatoll` (includes `-kilv`).
- If depmod warning about `modules.builtin.modinfo` appears during install: this is fixed in `debian/rules` via `patch_bootimage_postinst` which patches the generated postinst before packaging.

## Build log
- **Build 1** — 2026-07-07 22:52 UTC+8 (commit 82c30d9) — first build, stale out/ artifacts → kernel binary was old CI build, not ours
- **Build 2** — 2026-07-07 23:03 UTC+8 — rebuild with `rm -rf out/`, but `override` not yet added → kernel had `kilv@` build user but version still `4.14-336-xiaomi-miatoll`
- **Build 3** — 2026-07-07 23:59 UTC+8 (commit a037bc8) — added `override KERNEL_BASE_VERSION = 4.14-336-kilv` → kernel shows `4.14-336-kilv-xiaomi-miatoll (kilv@...)`
- **Build 4** — 2026-07-08 (commit 87d7023) — kernel fixes for monitor mode:
  - Fix 1 (`d610c12`): Added `sme_delete_mon_session()` before `hdd_vdev_destroy()` in `hdd_stop_adapter()` for `QDF_MONITOR_MODE` → fixes stale PE session table crash on ifup
  - Fix 2 (`87d7023`): Added `hdd_vdev_destroy()` after `hdd_deinit_adapter()` in change_iface MONITOR_MODE block
  - Result: `iw phy` path and `change_iface` with `iff_up=true` both survive 3 ifdown/ifup cycles
  - Remaining bug: `change_iface` with `iff_up=false` still panics (interface DOWN before conversion)

## QCACLD-3.0 monitor mode — preferred method

The `iw dev wlan0 set type monitor` (`change_iface`) path is unreliable for this driver, especially when the interface is DOWN before conversion. The **proper** method is the `con_mode` sysfs parameter, which performs a complete firmware idle shutdown/restart cycle:

```
# → monitor mode
svc wifi disable
ip link set wlan0 down
echo 4 > /sys/module/wlan/parameters/con_mode    # QDF_GLOBAL_MONITOR_MODE
ip link set wlan0 up

# → managed mode (STA)
ip link set wlan0 down
echo 0 > /sys/module/wlan/parameters/con_mode    # QDF_GLOBAL_MISSION_MODE
ip link set wlan0 up
svc wifi enable
```

Valid values:
- `0` — QDF_GLOBAL_MISSION_MODE (managed STA)
- `4` — QDF_GLOBAL_MONITOR_MODE (monitor)

Under the hood, `echo 4 > con_mode` calls `__hdd_driver_mode_change()` which:
1. `hdd_stop_present_mode()` — stops all adapters for current mode
2. `pld_idle_shutdown()` — firmware idle shutdown
3. `hdd_cleanup_present_mode()` — cleans up all state
4. `cds_set_conparam(next_mode)` — sets the new mode
5. `pld_idle_restart()` — firmware restart (full reinit)
6. `hdd_open_adapters_for_mode()` — opens adapters for new mode
7. `hdd_start_adapter()` — starts the monitor adapter

The `con_mode` approach avoids all the state consistency issues that plague the `change_iface` path.

## Kernel source changes for monitor mode fixes

Key files changed in commit 87d7023:

- **`drivers/staging/qcacld-3.0/core/hdd/src/wlan_hdd_main.c`**:
  - `hdd_stop_adapter()`: Added `sme_delete_mon_session(hdd_ctx->mac_handle, adapter->vdev_id)` before `hdd_vdev_destroy()` in the `QDF_MONITOR_MODE` case. This deletes the PE session table entry so `gpSession[vdev_id]` is no longer `valid=true` with dangling pointers.
  - `__wlan_hdd_cfg80211_change_iface()`: Added `hdd_vdev_destroy(adapter)` after `hdd_deinit_adapter()` in the `QDF_MONITOR_MODE` case.


## init_boot image (header_version 2 DTB fix)
- Reference: https://github.com/droidian/linux-packaging-snippets/blob/droidian/kernel-snippet.mk#L357
- The kernel-snippet.mk init_boot rule does NOT pass `--dtb` to mkbootimg, but `--header_version 2` requires it.
- Fix: override the pattern rule in `debian/rules` **BEFORE** the `include` statement:
  ```makefile
  out/KERNEL_OBJ/init_boot-%.img: out/KERNEL_OBJ/initramfs.% out/KERNEL_OBJ/dtb-merged
  	eval mkbootimg \
  		--header_version $(KERNEL_BOOTIMAGE_VERSION) \
  		--ramdisk $< \
  		--dtb $(KERNEL_OUT)/dtb-merged \
  		--dtb_offset $(KERNEL_BOOTIMAGE_DTB_OFFSET) \
  		--pagesize $(KERNEL_BOOTIMAGE_PAGE_SIZE) \
  		-o $@
  ```
- The override MUST be BEFORE `include /usr/share/linux-packaging-snippets/kernel-snippet.mk` because GNU Make searches pattern rules in definition order and uses the FIRST match. An override after the include is silently ignored.

## Kernel source
- repo: droidian-devices/linux-android-xiaomi-miatoll
- fork: kilvz/linux-android-xiaomi-miatoll
- branch: droidian
- defconfig: gnumdk_defconfig (used by build), NOT droidian_defconfig

## Debian packaging
- kernel-info.mk fields: keep ORIGINAL values (user reverted them)
- CLANG_CUSTOM = 1 must be set to prevent double-prepend of clang-android-() in toolchain
- Do NOT modify version or version-suffix fields
- To override a kernel-snippet.mk pattern rule in debian/rules: put the override BEFORE `include /usr/share/linux-packaging-snippets/kernel-snippet.mk`. GNU Make tries pattern rules in definition order; the first match wins. An override after the include is silently ignored.
