# Experiment Log

All experiments conducted on Raspberry Pi 4 Model B Rev 1.5 (BCM2711C0, 8GB)
with heatsink + 2 fans. OS: Debian 13 (trixie), kernel 6.18.29-rpt-v8.

## Config: `config.txt` (boot partition)

### Trial 1 — CPU 2.0 GHz + GPU 600 + over_voltage=2

**Settings:**
```
arm_freq=2000
gpu_freq=600
over_voltage=2
core_freq=550
temp_limit=85
```

**Result: -- System did not boot**

Pi4 became completely unresponsive -- no ping, no SSH.
Recovery: removed SD card, edited config.txt on Pi5 (another machine), removed overclock lines.
This chip cannot handle 2.0 GHz at only +0.05V over stock.

---

### Trial 2 — CPU 1.9 GHz + GPU 550 + over_voltage=4

**Settings:**
```
arm_freq=1900
gpu_freq=550
over_voltage=4
core_freq=525
temp_limit=85
```

**Result: -- Stable**

Booted successfully. Temperatures around 50C under light load.
Entry-level overclock -- safe for any Pi4.

---

### Trial 3 — CPU 2.0 GHz + GPU 600 + over_voltage=6

**Settings:**
```
arm_freq=2000
gpu_freq=600
over_voltage=6
core_freq=550
temp_limit=85
```

**Result: -- Stable**

2.0 GHz worked with over_voltage=6 (+0.15V). Temperature ~53C.
No throttling. This confirms this chip needs higher voltage for 2.0 GHz.

---

### Trial 4 — GPU 750 MHz (CPU 2.0 GHz unchanged)

**Settings:**
```
arm_freq=2000
gpu_freq=750
over_voltage=6
core_freq=550
temp_limit=85
```

**Result: -- Unstable after ~1 hour**

GPU V3D at 750 MHz works fine initially. Temperature ~57C. Browser UI snappier.
After about 1 hour of use, V3D starts hanging every second:

```
v3d fec00000.v3d: *ERROR* V3D_ERR_STAT: 0x00001000
v3d fec00000.v3d: *ERROR* Resetting GPU for hang.
```

The driver resets the GPU, but the hang loop continues indefinitely.
Occurs under real workloads (browser, WebGL), not during idle.

---

### Trial 5 — GPU 850 MHz + Core 600 MHz (limit test)

**Settings:**
```
arm_freq=2000
gpu_freq=850
over_voltage=6
core_freq=600
temp_limit=85
```

**Result: -- Unstable**

- GPU V3D: confirmed 850 MHz
- Core: showed 425 MHz (setting did not apply -- driver limitation)
- YouTube video page failed to load -- GPU video decoder (H264/ISP) unstable
- USB errors in dmesg: cannot get freq at ep 0x82 -- core_freq=600 caused USB instability
- **System hung completely** -- hardware watchdog (1 min timeout) force-rebooted

---

### Trial 6 — GPU 600 MHz + over_voltage=4 (labwc stability fix)

**Date:** 2026-05-27

**Settings:**
```
arm_freq=2000
gpu_freq=600
over_voltage=4
core_freq=550
temp_limit=85
# v3d_freq not set — inherits gpu_freq=600
```

**Context:** After prolonged use, labwc (Wayland compositor) would occasionally freeze
on dual-display setup (DSI 4.3" + HDMI). Suspected cause: over_voltage=6 with
v3d_freq=700 pushing GPU voltage regulator to instability zone.

**Result: -- Fully stable (new final)**

- CPU 2.0 GHz, GPU 600 MHz, V3D inherits 600 MHz
- labwc: no freezes after 24h+ of testing (previously froze every few hours)
- YouTube 1080p: smooth playback after reboot
- Temperature under load: ~60C
- No throttling (throttled=0x0)
- PSU noise immunity: verified on UPS 5V 3A (2×18650)

### Final (revised) — GPU 600 MHz + Core 550 MHz + over_voltage=4 (stable workstation)

**Settings:**
```
arm_freq=2000
gpu_freq=600
over_voltage=4
core_freq=550
temp_limit=85
# v3d_freq removed — inherits gpu_freq=600
```

**Result: -- Fully stable, labwc no longer freezes**

- CPU 2.0 GHz, GPU 600 MHz, Core 550 MHz
- V3D hang count: 0 (no GPU resets)
- labwc: stable on DSI + HDMI dual display
- YouTube 1080p, browser, all apps work
- Temperature under full load (CPU 100% + V3D): ~60C
- No throttling (throttled=0x0)
- **UPS 5V (2×18650):** verified stable — all settings work within UPS power budget

## Key findings

1. **This Pi4 rev 1.5 needs over_voltage=6 for 2.0 GHz** -- ov=2 was insufficient
2. **GPU max stable frequency: 700 MHz** -- 750 MHz eventually hangs V3D under load
3. **core_freq=600 destabilizes USB** -- error: "cannot get freq at ep 0x82"
4. **Hardware watchdog** (bcm2835-wdt, 1 min timeout) automatically reboots on complete hang
5. **V3D hang test**: `dmesg | grep -c 'Resetting GPU for hang'` -- should be 0

## Benchmark (informal)

| Test | Stock (1.8 GHz) | Overclocked (2.0 GHz + 700 GPU) | Improvement |
|------|----------------|-------------------------------|-------------|
| Browser tab open | 1.8s | 1.4s | ~22% |
| YouTube page load | 3.2s | 2.5s | ~22% |
| UI responsiveness | baseline | noticeably smoother | -- |
