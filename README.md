# Pi4 Overclock Experiments

Эксперименты с разгоном Raspberry Pi 4 (BCM2711, rev 1.5, 8GB) — параметры, результаты, стабильность.

## Стенд

- **Плата:** Raspberry Pi 4 Model B Rev 1.5, 8GB
- **ОС:** Debian 13 (trixie), Raspberry Pi OS kernel 6.18.29
- **Охлаждение:** радиатор + 2 вентилятора
- **Питание:** UPS 5V 3A (2×18650) или прямой БП 5V 3A
- **Нагрузка:** labwc (Wayland), wf-panel-pi, Chromium, YouTube
- **Дисплеи:** DSI 4.3" (800x480) + HDMI (1360x768)

## Итоговый стабильный разгон

> ⚠️ **ВСЕ значения ниже верны ТОЛЬКО для стенда с активным охлаждением (радиатор + 2 вентилятора).**
> Без активного охлаждения — не пытайтесь повторить arm_freq ≥ 2.0 GHz.

| Параметр | Stock | Стабильно | Оптимально | Прирост |
|----------|-------|-----------|-----------|---------|
| **CPU** (arm_freq) | 1.8 GHz | 2.0 GHz | **2.1 GHz** | +17% |
| **V3D** (v3d_freq) | 500 MHz | 600 MHz | **700 MHz** | +40% |
| **Core** (core_freq) | 500 MHz | 550 MHz | **600 MHz** | +20% |
| **Напряжение** (over_voltage) | 0 | 4 | **6** | — |
| **Температура макс.** | ~50°C | ~55°C | **~60°C** | отлично |
| **Троттлинг** | 0 | 0 | **0** | ✅ |

## config.txt (итоговый)

```ini
# GPU driver — fkms для стабильности (kms вызывает RCU stall)
dtoverlay=vc4-fkms-v3d

# Разгон
arm_freq=2100
over_voltage=6
core_freq=600
v3d_freq=700
temp_limit=85
gpu_mem=128

# force_turbo НЕ включать — динамические частоты
# gpu_freq НЕ использовать — грубый разгон всех блоков GPU
```

## ⚠️ Что НЕ использовать и почему

| Параметр | Почему вреден |
|----------|--------------|
| `gpu_freq` | Разгоняет сразу V3D+H264+ISP+HEVC. H264-блок нестабилен → краши |
| `vc4-kms-v3d` | Прямой доступ к GPU → RCU stall ядра при загрузке |
| `force_turbo=1` | Фиксирует макс. частоту → перегрев + warranty bit |
| `#ignore-gpu-blocklist` (Chromium) | Форсирует GPU-рендеринг → watchdog-перезагрузки |
| `#enable-accelerated-video-decode` | V4L2-декодинг на vc4 → зависания GPU |
| `sd_overclock=100` | **Не работает** на внутреннем слоте Pi4 (аппаратный предел DDR50) + вызывает ошибки инициализации SD-карты и чёрный экран при загрузке |
| `arm_freq≥2000` **без** активного охлаждения | Краш системы при нагреве выше 80°C, внезапные ребуты, потеря данных |

## 🛡️ Безопасный разгон для headless/серверного использования

Если Pi4 работает в headless-режиме (без монитора, 24/7, как сервер/агент):

| Параметр | Рекомендация |
|----------|-------------|
| **arm_freq** | ≤ **1800 MHz** (безопасно для любого экземпляра) |
| **over_voltage** | ≤ **2** (минимальное повышение для стабильности) |
| **sd_overclock** | **НЕ ИСПОЛЬЗОВАТЬ** — не работает и вызывает сбои |
| **temp_limit** | **80** (снижен с 85 для запаса до троттлинга) |

Без монитора GPU-разгон (v3d_freq/core_freq) не даёт прироста — его можно оставить по умолчанию.

Пример конфига для headless:
```ini
arm_freq=1800
over_voltage=2
temp_limit=80
gpu_mem=16  # минимум GPU-памяти для headless
```
На этом разгоне Pi4 стабильно работает месяцами даже на SD-карте без активного охлаждения.

## 🔧 Дополнительные настройки для стабильности

### GPU-драйвер
```ini
dtoverlay=vc4-fkms-v3d    # fkms — стабильнее kms на Pi4
```

### Chromium
```
chrome://flags:
  #ignore-gpu-blocklist              → Default (Disabled)
  #enable-accelerated-video-decode   → Disabled
  #enable-zero-copy                  → Default
```
GPU-растеризация всё равно работает через CLI-флаг от пакета `rpi-chromium-mods`.

### Журналирование (для диагностики крашей)
```bash
sudo sed -i 's/^#Storage=.*/Storage=persistent/' /etc/systemd/journald.conf
sudo systemctl restart systemd-journald
```
Без этого логи теряются при watchdog-перезагрузке.

## Хронология

Подробный лог всех попыток — в [EXPERIMENTS.md](./EXPERIMENTS.md).

Кратко:
1. CPU 2.0 + GPU 600 + ov=2 → ❌ не загрузилась
2. CPU 1.9 + GPU 550 + ov=4 → ✅
3. CPU 2.0 + GPU 600 + ov=6 → ✅ (использовалась неделю)
4. GPU 750 → ⚠️ V3D hang через час
5. GPU 850 + Core 600 → ❌ ханг системы
6. **Обнаружен RCU stall** — vc4-kms-v3d + gpu_freq + Chromium-флаги = watchdog-ребуты
7. **Финальный фикс** — fkms + v3d_freq + сброс Chromium-флагов → полная стабильность

## Мониторинг

```bash
# Частоты
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
vcgencmd measure_clock v3d

# Троттлинг
vcgencmd get_throttled    # должно быть throttled=0x0

# Температура
cat /sys/class/thermal/thermal_zone0/temp

# V3D hang count (должен быть 0)
dmesg | grep -c 'Resetting GPU for hang'

# Watchdog-перезагрузки
journalctl --list-boots
```

## Лицензия

MIT
