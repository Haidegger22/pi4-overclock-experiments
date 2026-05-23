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

**Result: ❌ System did not boot**

Pi4 became completely unresponsive — no ping, no SSH. 
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

**Result: ✅ Stable**

Booted successfully. Temperatures around 50°C under light load.
Entry-level overclock — safe for any Pi4.

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

**Result: ✅ Stable**

2.0 GHz worked with over_voltage=6 (+0.15V). Temperature ~53°C.
No throttling. This confirms this chip needs higher voltage for 2.0 GHz.

**Observation:** Initial 2.0 GHz + ov=2 failed, but 2.0 GHz + ov=6 passed.
The chip is a "cold" sample that needs more voltage.

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

**Result: ✅ Stable**

GPU bumped from 600 to 750 MHz. Temperature rose to ~57°C.
Browser UI rendering noticeably snappier. No instability.

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

**Result: ❌ Unstable**

- GPU V3D: confirmed 850 MHz
- Core: showed 425 MHz (setting did not apply — possibly driver limitation)
- YouTube video page failed to load — GPU video decoder (H264/ISP) unstable
- USB errors in dmesg: `cannot get freq at ep 0x82` — core_freq=600 caused USB instability
- **System hung completely** — hardware watchdog (1 min timeout) force-rebooted

**Lessons learned:**
- 850 MHz is too high for the video decoder (H264/ISP) which shares `gpu_freq`
- core_freq=600 destabilizes the USB controller on this chip
- The hardware watchdog (bcm2835-wdt) saved us from a dead hang

---

### Final — GPU 750 MHz + Core 550 MHz (stable workstation)

**Settings:**
```
arm_freq=2000
gpu_freq=750
over_voltage=6
core_freq=550
temp_limit=85
```

**Result: ✅ Fully stable**

- CPU 2.0 GHz, GPU 750 MHz, Core 550 MHz
- YouTube, browser, all apps work
- Temperature under full load (CPU 100% + V3D): ~68°C
- No throttling (throttled=0x0)
- Cooler than stock under load
- UPS battery life impact: ~15% reduction (acceptable)

## Benchmark (informal)

| Test | Stock (1.8 GHz) | Overclocked (2.0 GHz + 750 GPU) | Improvement |
|------|----------------|-------------------------------|-------------|
| Browser tab open | 1.8s | 1.4s | ~22% |
| YouTube page load | 3.2s | 2.5s | ~22% |
| UI responsiveness | baseline | noticeably smoother | — |
