# AMD RX 7700 XT (Navi 32) GPU Freezes on Ubuntu 25.10 / Kernel 6.17

**Date Filed:** November 25, 2025
**Status:** Suspected (Under Investigation)
**Severity:** Critical (System Lockup)
**Source:** [Original Gist](https://gist.github.com/danielrosehill/965423a18f56a6d8ade74d616c2d97e1)

---

## Summary

Full system lockups (hard freeze, requiring power cycle) affecting AMD Radeon RX 7700 XT on Ubuntu 25.10 with kernel 6.17. Issue began after upgrading from Ubuntu 25.04 to 25.10 and appears to be a kernel regression affecting RDNA3 GPUs.

---

## System Configuration

| Component | Details |
|-----------|---------|
| **GPU** | AMD Radeon RX 7700 XT / 7800 XT (Navi 32, gfx1101) |
| **OS** | Ubuntu 25.10 |
| **Desktop** | KDE Plasma (Wayland) |
| **Kernel** | 6.17.0-6-generic (issue present), 6.14.0-15-generic (testing) |
| **CPU** | Intel Core i7-12700F |
| **RAM** | 64 GB |
| **VRAM** | 12 GB GDDR6 |

---

## Symptoms

- **Full system lockups** (hard freeze, requires power cycle)
- Random occurrence - not tied to specific GPU load
- Started after upgrading from Ubuntu 25.04 to 25.10 (kernel 6.14 → 6.17)
- No recovery possible once frozen (SysRq keys unresponsive)

---

## Logs & Errors

### Recurring DRM Scheduler Errors

```
[drm:drm_sched_entity_push_job [gpu_sched]] *ERROR* Trying to push to a killed entity
```

These appear frequently in syslog, often in pairs, throughout normal operation.

### Display Manager IRQ Warning (Boot)

```
workqueue: dm_irq_work_func [amdgpu] hogged CPU for >10000us 4 times, consider switching to WQ_UNBOUND
```

### SMU Driver Version Mismatch

```
amdgpu 0000:03:00.0: amdgpu: smu driver if version = 0x0000003d, smu fw if version = 0x00000040
amdgpu 0000:03:00.0: amdgpu: SMU driver if version not matched
```

### Other Notable Messages

```
amdgpu 0000:03:00.0: amdgpu: DF poison setting is inconsistent(1:0:0:0)!
amdgpu 0000:03:00.0: amdgpu: Poison setting is inconsistent in DF/UMC(0:1)!
amdgpu: Freeing queue vital buffer 0x..., queue evicted
```

---

## Kernel Parameters Tested

### Current GRUB Configuration (kernel 6.17)

```
amdgpu.gpu_recovery=1
amdgpu.ppfeaturemask=0xfffd7fff
amdgpu.dpm=1
amdgpu.runpm=0
amdgpu.dc=1
amdgpu.dcfeaturemask=0x8
amdgpu.tmz=0
amdgpu.sg_display=0
amdgpu.dcdebugmask=0x10
iommu=soft
```

### Previously Tested (kernel 6.14)

```
amdgpu.dpm=1
amdgpu.gpu_recovery=1
amdgpu.vm_fragment_size=9
amdgpu.lockup_timeout=60000
amdgpu.noretry=0
amdgpu.ppfeaturemask=0xffffffff
```

---

## Debugging Steps Taken

### 1. Removed Custom Power Limit Services

Had two conflicting systemd services limiting GPU power:
- `amdgpu-power-limit.service` → 220W
- `gpu-power-limit.service` → 180W

**Resolution**: Removed both services, reset to default 200W power cap.

```bash
sudo systemctl disable --now amdgpu-power-limit.service gpu-power-limit.service
sudo rm /etc/systemd/system/amdgpu-power-limit.service /etc/systemd/system/gpu-power-limit.service
echo 200000000 | sudo tee /sys/class/drm/card1/device/hwmon/hwmon*/power1_cap
```

### 2. Downgraded to Kernel 6.14

Set kernel 6.14.0-15-generic as default boot option to test if issue is a 6.17 regression:

```bash
sudo sed -i 's|GRUB_DEFAULT=.*|GRUB_DEFAULT="gnulinux-advanced-UUID>gnulinux-6.14.0-15-generic-advanced-UUID"|' /etc/default/grub
sudo update-grub
```

**Status**: Testing in progress

---

## Related Resources

- [RDNA3 TLB Fence Issues Guide](https://gist.github.com/danielrosehill/6a531b079906f160911a87dea50e1507)
- [Arch Forums - System freeze because of amdgpu driver](https://bbs.archlinux.org/viewtopic.php?id=302000)
- [Arch Forums - AMD GPU crashing (SOLVED)](https://bbs.archlinux.org/viewtopic.php?id=299954)
- [freedesktop.org Bug #111481 - AMD Navi GPU frequent freezes](https://bugs.freedesktop.org/show_bug.cgi?id=111481)

---

## Known Issues (Kernel 6.14-6.17 + RDNA3)

Per community reports, there's a documented kernel bug in memory management affecting RX 7000 series GPUs on kernels 6.14-6.17, causing:
- GPU freezes
- System hangs
- "Fence timeout" errors in logs

---

## Next Steps

1. Test stability on kernel 6.14
2. If stable on 6.14, monitor upstream for 6.17+ fixes
3. Consider testing `amdgpu.gfx_off=0` parameter if issues persist
4. File bug report with AMD/kernel team if reproducible case identified

---

## Hardware Details

```
amdgpu 0000:03:00.0: amdgpu: initializing kernel modesetting (IP DISCOVERY 0x1002:0x747E 0x1DA2:0x475F 0xFF)
VRAM: 12272M (12272M used)
GART: 512M
RAM width 192bits GDDR6
SE 3, SH per SE 2, CU per SH 10, active_cu_number 54
```
