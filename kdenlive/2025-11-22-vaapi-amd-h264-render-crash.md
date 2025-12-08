# Kdenlive 25.08.3 VAAPI AMD H264 Render Crash

**Date Filed:** November 22, 2025
**Status:** Confirmed
**Severity:** High (Application Crash)
**Source:** [Original Gist](https://gist.github.com/danielrosehill/afa8b24aaae92a4f7160667facc3e9ee)

---

## Summary

Kdenlive crashes when attempting to render using the **VAAPI AMD H264** hardware encoding profile. VAAPI is correctly configured and functional on the system, and direct FFmpeg VAAPI encoding works inside the Flatpak sandbox. The issue appears to be a memory management problem between Kdenlive's MLT renderer and the AMD GPU driver.

---

## Environment

| Component | Details |
|-----------|---------|
| **OS** | Ubuntu 25.04 |
| **Desktop** | KDE Plasma (Wayland) |
| **Kdenlive** | 25.08.3 (Flatpak from Flathub, stable branch) |
| **GPU** | AMD Radeon RX 7700 XT (Navi 32, gfx1101) |
| **Kernel** | 6.14.0-15-generic |
| **Mesa** | 25.2.3-1ubuntu1 |
| **libva** | 2.22.0 (VA-API 1.22) |

---

## Symptoms

- Kdenlive crashes during render when VAAPI AMD H264 profile is selected
- Segfault in `kdenlive_render` process
- GPU memory cleanup errors in dmesg

---

## VAAPI Status

VAAPI is correctly detected and functional on the system:

```
$ vainfo
libva info: VA-API version 1.22.0
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri/radeonsi_drv_video.so
libva info: Found init function __vaDriverInit_1_22
libva info: va_openDriver() returns 0
Trying display: wayland
vainfo: VA-API version: 1.22 (libva 2.22.0)
vainfo: Driver version: Mesa Gallium driver 25.2.3-1ubuntu1 for AMD Radeon RX 7700 XT (radeonsi, navi32, LLVM 20.1.8, DRM 3.61, 6.14.0-15-generic)
vainfo: Supported profile and entrypoints
      VAProfileH264ConstrainedBaseline: VAEntrypointVLD
      VAProfileH264ConstrainedBaseline: VAEntrypointEncSlice
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointEncSlice
      VAProfileH264High               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointEncSlice
      VAProfileHEVCMain               : VAEntrypointVLD
      VAProfileHEVCMain               : VAEntrypointEncSlice
      VAProfileHEVCMain10             : VAEntrypointVLD
      VAProfileHEVCMain10             : VAEntrypointEncSlice
      VAProfileJPEGBaseline           : VAEntrypointVLD
      VAProfileVP9Profile0            : VAEntrypointVLD
      VAProfileVP9Profile2            : VAEntrypointVLD
      VAProfileAV1Profile0            : VAEntrypointVLD
      VAProfileAV1Profile0            : VAEntrypointEncSlice
      VAProfileNone                   : VAEntrypointVideoProc
```

---

## VAAPI Works in Flatpak FFmpeg

Direct FFmpeg VAAPI encoding works correctly inside the Kdenlive Flatpak sandbox:

```
$ flatpak run --command=sh org.kde.kdenlive -c "ffmpeg -vaapi_device /dev/dri/renderD128 -f lavfi -i testsrc=duration=1:size=1280x720:rate=30 -vf 'format=nv12,hwupload' -c:v h264_vaapi -f null -"

[h264_vaapi @ 0x56baf78e1280] No quality level set; using default (20).
Output #0, null, to 'pipe:':
  Stream #0:0: Video: h264 (High), vaapi(tv, progressive), 1280x720 [SAR 1:1 DAR 16:9], q=2-31, 30 fps, 30 tbn
frame=   30 fps=0.0 q=-0.0 Lsize=N/A time=00:00:00.96 bitrate=N/A speed=14.5x
```

---

## Crash Logs (dmesg)

```
[ 4216.375975] Thread (pooled)[855503]: segfault at 48bffc894954 ip 00007734b73831e2 sp 00007733b7ffe338 error 4 in libQt6Qml.so.6.9.3[3831e2,7734b70eb000+49e000] likely on CPU 4 (core 8, socket 0)
[ 4359.985692] amdgpu 0000:03:00.0: amdgpu: VM memory stats for proc melt(892791) task melt:cs0(892595) is non-zero when fini
[ 4359.986033] kdenlive_render[892010]: segfault at 7145e0021008 ip 00007145e98aa6e2 sp 00007ffe5d29feb0 error 4 in libc.so.6[a96e2,7145e9829000+175000] likely on CPU 9 (core 16, socket 0)
```

---

## Analysis

1. **`kdenlive_render` segfaults in libc.so.6** - the render process crashes
2. **`amdgpu: VM memory stats... is non-zero when fini`** - GPU memory not properly cleaned up by `melt` (MLT render backend)
3. **Qt6Qml segfault** - likely secondary crash from UI thread

This appears to be a memory management issue between Kdenlive's MLT renderer and the AMD GPU driver when using VAAPI encoding.

---

## Workarounds

- Use software encoding (libx264/libx265) instead of VAAPI
- Possibly try VAAPI AV1 encoding (different code path)

---

## Flatpak Info

```
Kdenlive - Video editor

          ID: org.kde.kdenlive
         Ref: app/org.kde.kdenlive/x86_64/stable
        Arch: x86_64
      Branch: stable
     Version: 25.08.3
     License: GPL-3.0-only
      Origin: flathub
  Collection: org.flathub.Stable
Installation: system
   Installed: 236.6 MB
     Runtime: org.kde.Platform/x86_64/6.9
         Sdk: org.kde.Sdk/x86_64/6.9

      Commit: ec9c64d3576c55d159d4f307b13fa01b0e41ea60bbd29cd7ae3fc0d1220457cf
     Subject: update kdenlive 25.08.3 (70a6dc04c150)
        Date: 2025-11-22 02:12:49 +0000
```
