# Experiment Log

All experiments on Raspberry Pi 4 Model B Rev 1.5 (BCM2711C0, 8GB)
with heatsink + 2 fans. OS: Debian 13 (trixie), kernel 6.18.29-rpt-v8.

## Config: `config.txt` (boot partition)

### Trial 1 — CPU 2.0 GHz + GPU 600 + over_voltage=2

**Settings:** arm_freq=2000, gpu_freq=600, over_voltage=2, core_freq=550

**Result: ❌ System did not boot**

No ping, no SSH. Recovery: removed SD, edited on Pi5.

---

### Trial 2 — CPU 1.9 GHz + GPU 550 + over_voltage=4

**Settings:** arm_freq=1900, gpu_freq=550, over_voltage=4, core_freq=525

**Result: ✅ Stable** — ~50°C, safe for any Pi4.

---

### Trial 3 — CPU 2.0 GHz + GPU 600 + over_voltage=6

**Settings:** arm_freq=2000, gpu_freq=600, over_voltage=6, core_freq=550

**Result: ✅ Stable** — ~53°C, no throttling.

---

### Trial 4 — GPU 750 MHz

**Settings:** arm_freq=2000, gpu_freq=750, over_voltage=6, core_freq=550

**Result: ⚠️ Unstable after ~1h**

```
v3d fec00000.v3d: *ERROR* V3D_ERR_STAT: 0x00001000
v3d fec00000.v3d: *ERROR* Resetting GPU for hang.
```

---

### Trial 5 — GPU 850 MHz + Core 600 MHz (limit test)

**Settings:** arm_freq=2000, gpu_freq=850, over_voltage=6, core_freq=600

**Result: ❌ System hung** — USB errors, GPU crash, watchdog reboot.

---

### Trial 6 — GPU 600 + over_voltage=4 (labwc fix, old "final")

**Date:** 2026-05-27

**Settings:** arm_freq=2000, gpu_freq=600, over_voltage=4, core_freq=550

**Result: ✅ Fully stable** — but later discovered gpu_freq is the wrong approach.

---

### Trial 7 — RCU Stall & Watchdog Reboots (DISCOVERY)

**Date:** 2026-05-27

**Settings at time:** arm_freq=2000, gpu_freq=600, over_voltage=4, core_freq=550, vc4-kms-v3d

**Symptoms:** System randomly froze or rebooted every few hours. SSH/ping dead. Screen showed RCU stall:
```
rcu_preempt detected stalls on CPUs/tasks
rcu_preempt kthread starved for 5823 jiffies
Possible timer handling issue on cpu=0
```

**Root cause:** Three interacting factors:

1. **vc4-kms-v3d driver** — direct GPU access can deadlock kernel during init or under load
2. **gpu_freq=600** — overclocks ALL GPU blocks (V3D + H264 + ISP + HEVC). H264/HEVC are unstable when overclocked
3. **Chromium #ignore-gpu-blocklist=Enabled** — forces Chromium to use vc4 GPU despite it being in blocklist

**Fix:** vc4-kms-v3d → vc4-fkms-v3d, gpu_freq → v3d_freq=600, Chromium flags reset to default.

---

### Trial 8 — Optimal: fkms + v3d_freq + Chromium flags fixed

**Date:** 2026-05-28

**Settings:**
```
arm_freq=2100
over_voltage=6
core_freq=600
v3d_freq=700
temp_limit=85
gpu_mem=128
# GPU driver: dtoverlay=vc4-fkms-v3d
# Chromium: all flags default EXCEPT arm_boost
```

**Result: ✅ Fully stable**

- No RCU stalls, no watchdog reboots
- Temperature under load: ~60°C
- No throttling (throttled=0x0)
- Dual display (DSI + HDMI): stable
- YouTube 1080p: smooth
- 24h+ uptime verified

---

## Key Findings

1. **gpu_freq is DANGEROUS** — overclocks too many blocks at once (V3D+H264+ISP+HEVC). Use `v3d_freq` instead
2. **vc4-kms-v3d causes RCU stall** → use `vc4-fkms-v3d` for stability
3. **Chromium #ignore-gpu-blocklist crashes Pi4** — keep Disabled. Let rpi-chromium-mods handle GPU rasterization via CLI flag
4. **#enable-accelerated-video-decode crashes Pi4** — V4L2 decoding on vc4 is unstable
5. **Hardware watchdog** (bcm2835-wdt, 1min) force-reboots on complete hang — not a fix, just a symptom
6. **Pi4 SD controller maxes at DDR50** (50MHz) — `sd_overclock=100` doesn't apply to internal slot
7. **UPS 5V 3A (2×18650) verified stable** for all tested configs
8. **Direct PSU 5V 3A (no UPS) also stable** for optimal config

## Benchmark

| Test | Stock (1.8 GHz) | Optimal (2.1 GHz + v3d 700) | Gain |
|------|----------------|---------------------------|------|
| Browser tab | 1.8s | 1.2s | ~33% |
| YouTube load | 3.2s | 2.1s | ~34% |
| UI feel | baseline | much smoother | — |
