# OxiKernel — Exynos 9610 (Galaxy A50 / SM-A505F) Android 16 Release

**Date:** 2026-08-13  
**Kernel base:** Linux 4.14.336 (Android common, AArch64)  
**Target device:** Samsung Galaxy A50 (Exynos 9610 / universal9610)  
**Target system:** Android 16 GSI / LineageOS 23.2  
**Compiler:** Proton Clang 13 + LLD  
**SELinux:** Enforcing (optional: permissive via `build.sh --permissive`)  

---

## 🚀 Why OxiKernel?

Stock Samsung Exynos9610 kernels are **incompatible with Android 14/15/16 GSIs**:
- `CONFIG_SEC_REBOOT` disabled → power button reboots instead of shutting down, `reboot recovery` does nothing
- No inline encryption support → modern FBE / metadata encryption cannot work
- Android 11-level SELinux / VINTF → kernel panic on Android 16 init
- Missing 64-bit binder v8, binderfs, EROFS, overlay, and other Android 14+ requirements

OxiKernel fixes all of this by **backporting from reference kernels** (Motorola exynos9610-lineage-23.2, Samsung mint-android-14.0) into a single flashable package.

---

## ✨ What's New in This Release

| Feature | Status | Description |
|---------|--------|-------------|
| **SEC_REBOOT** | ✅ Active | Registers `pm_power_off` + `arm_pm_restart`. Power button now actually powers off; `reboot recovery` works |
| **SYSVIPC + IPC_NS** | ✅ Active | Android 16 GSI IPC namespaces support |
| **ENCRYPTED_KEYS** | ✅ Active | FBE / metadata encryption keyring |
| **Inline Crypto Stack** | ✅ Backported | From Motorola 4.14.357-openela: `dm-default-key`, `blk-crypto`, `bio-crypt-ctx`, `keyslot-manager`, `inline_crypt` |
| **BLK_INLINE_ENCRYPTION** | ✅ Active | Hardware-agnostic inline encryption fallback |
| **DM_BOW** | ✅ Active | Backup block device |
| **64-bit Binder v8** | ✅ Active | `CONFIG_ANDROID_BINDER_IPC=y`, `BINDER_CURRENT_PROTOCOL_VERSION=8` |
| **Binderfs** | ✅ Active | Modern Android binder mount model |
| **EROFS + squashfs** | ✅ Active | GSI system / vendor mount |
| **OverlayFS** | ✅ Active | GSI overlay support |
| **Incremental FS** | ✅ Active | Play Store streaming / app streaming |
| **dm-verity + AVB** | ✅ Active | Boot verification |
| **ION + Exynos CMA** | ✅ Active | GPU / camera / display heaps |
| **Pstore / Ramoops** | ✅ Active | Crash log persistence |
| **Android 16 namespaces** | ✅ Active | USER_NS, UTS_NS, PID_NS, NET_NS |

---

## 📦 Package Contents

```
OxiKernel-<date>-enforcing.zip
├── boot.img          (OxiKernel + dtb_exynos.img, ~42 MB)
├── dtb.img           (Device Tree Blob for Exynos 9610)
├── dtbo.img          (Device Tree Blob Overlay for Exynos 9610)
└── META-INF/         (Android boot image header, etc.)
```

---

## 🔧 Flash Instructions

### Prerequisites
- TWRP (existing) is booted
- Android 16 GSI / LineageOS 23.2 `system.img` ready (this package is kernel only; system is separate)

### Steps

1. **TWRP > Wipe > Advanced** → check Cache + Dalvik → Swipe to Wipe

2. **TWRP > Install**  
   → select `OxiKernel-<date>-enforcing.zip`  
   → Swipe to Confirm

3. (Optional) **TWRP > Install**  
   → select LineageOS 23.2 `system.img` or Android 16 GSI `system.img`  
   → Swipe to Confirm

4. **TWRP > Reboot > System**

---

## ⚠️ Warnings & Notes

### SELinux
- Default is **Enforcing**. If boot hangs, try `build.sh --permissive` to build a permissive variant for debugging.

### Kernel Logs (Debug)
If boot hangs, use TWRP terminal:
```bash
dmesg > /tmp/dmesg.txt
cat /tmp/dmesg.txt | grep -iE "panic|bug|error|selinux|fstab|mount"
```

### Building from Source
```bash
./build.sh --enforcing    # SELinux enforcing
./build.sh --permissive   # SELinux permissive (debug)
```

---

## 🛠️ Fixes Applied (Technical Deep Dive)

### 1. SEC_REBOOT
**Problem:** `CONFIG_SEC_REBOOT` disabled → `sec_reboot.c` not compiled → no `pm_power_off` / `arm_pm_restart` registration.  
**Impact:** Power button triggers reboot instead of shutdown; `reboot recovery` does nothing.  
**Fix:** `kernel/configs/oxikernel_gsi-16.config` → `CONFIG_SEC_REBOOT=y`  
**Reference:** `drivers/samsung/sec_reboot.c:240-245` (`subsys_initcall` registration)

### 2. Inline Crypto Stack (Kernel Backport)
**Problem:** Samsung FMP (`BLK_DEV_CRYPT`) is legacy; Android 16 FBE requires generic inlinecrypt support.  
**Fix:** Backported from Motorola exynos9610-lineage-23.2 (4.14.357-openela):
- `block/blk-crypto.c`, `blk-crypto-fallback.c`, `blk-crypto-internal.h`, `bio-crypt-ctx.c`, `keyslot-manager.c`
- `drivers/md/dm-default-key.c`
- `fs/crypto/inline_crypt.c`
- `include/linux/blk-crypto.h`, `keyslot-manager.h`, `bio-crypt-ctx.h`
- `struct bio`, `struct request_queue`, `struct dm_target` header additions
- Makefile / Kconfig integration

### 3. Binder v8 + Binderfs
**Problem:** Stock kernel uses 32-bit binder protocol v7 or lacks binderfs. Android 16 GSI requires 64-bit binder v8 + binderfs.  
**Fix:** `CONFIG_ANDROID_BINDER_IPC=y`, `CONFIG_ANDROID_BINDERFS=y`, `CONFIG_ANDROID_BINDER_DEVICES="binder,hwbinder,vndbinder"`

### 4. VINTF / Android 16 Readiness
**Problem:** Stock kernel lacks Android 14+ filesystem, namespace, and security features required by GSI.  
**Fix:** Enabled EROFS, OverlayFS, Incremental FS, dm-verity + AVB, ION, namespaces, seccomp, and all Android 16 GSI mandatory options via `kernel/configs/oxikernel_gsi-16.config`.

---

## 📊 Version Compatibility

| Android Version | Kernel 4.14 | Boots |
|----------------|------------|-------|
| 11 (SDK 30) | ✅ | ✅ |
| 12 (SDK 31) | ✅ | ✅ |
| 13 (SDK 33) | ✅ | ✅ |
| 14 (SDK 34) | ✅ | ✅ |

---

## 🔗 References

- **Kernel base:** [Samsung exynos9610 mint-android-14.0](https://github.com/…) (reference: SEC_REBOOT, exynos-reboot)
- **Inline crypto backport source:** [Motorola exynos9610-lineage-23.2 (4.14.357-openela)](https://github.com/…) (dm-default-key, blk-crypto, keyslot-manager)
- **Device tree reference:** [android_device_motorola_exynos9610-common-lineage-23.2](https://github.com/…) (fstab, VINTF manifest structure)
- **Vendor patch tool (separate):** `vendor-patch/patch_vendor.sh` (Knox / FBE / WSM / CASS removal)

---

## 🐛 Known Issues

- `dm-default-key` backport is generic; Exynos 9610 ICE hardware (if present) is not optimized (fallback mode active)
- Some Samsung-specific HALs (camera.exynos9610.so, gralloc.exynos9610.so) may conflict with GSI HALs — use a compatible GSI / LineageOS build

---

## 📝 License

OxiKernel kernel code is GPLv2.
