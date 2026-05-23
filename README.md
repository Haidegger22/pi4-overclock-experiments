# Pi4 Overclock Experiments

Эксперименты с разгоном Raspberry Pi 4 (BCM2711, rev 1.5) — параметры, результаты, стабильность.

## Стенд

- **Плата:** Raspberry Pi 4 Model B Rev 1.5
- **ОС:** Debian 13 (trixie), Raspberry Pi OS kernel 6.18.29
- **Охлаждение:** радиатор + 2 вентилятора
- **Блок питания:** UPS 5V 3A (2×18650)
- **Нагрузка:** labwc (Wayland), wf-panel-pi, браузер (вкладки, YouTube)
- **Дисплеи:** DSI 4.3" (800×480) + HDMI (1360×768)

## Сводка настроек

| Компонент | Stock | Итоговый разгон | Прирост |
|-----------|------|----------------|---------|
| **CPU** (arm_freq) | 1.8 GHz | **2.0 GHz** | +11% |
| **GPU** (gpu_freq) | 500 MHz | **750 MHz** | +50% |
| **Core** (core_freq) | 500 MHz | **550 MHz** | +10% |
| **Напряжение** (over_voltage) | 0 | **6** | безопасно |
| **Температура макс** (100% CPU+GPU) | ~50°C | **~68°C** | отлично |
| **Троттлинг** | 0 | **0** | ни разу |

## config.txt (итоговый рабочий)

```ini
arm_freq=2000
gpu_freq=750
over_voltage=6
core_freq=550
temp_limit=85
```

## Хронология экспериментов

### Этап 1: CPU 2.0 GHz + GPU 600 + over_voltage=2
```ini
arm_freq=2000    ❌ — система не загрузилась
gpu_freq=600
over_voltage=2
core_freq=550
```
**Результат:** не загрузилась. SD-карта вынута, config.txt почищен на Pi5.

### Этап 2: CPU 1.9 GHz + GPU 550 + over_voltage=4
```ini
arm_freq=1900    ✅ — стабильно
gpu_freq=550
over_voltage=4
core_freq=525
temp_limit=85
```
**Результат:** загрузилась. Температура ~50°C. Работает.

### Этап 3: CPU 2.0 GHz + GPU 600 + over_voltage=6
```ini
arm_freq=2000    ✅ — взлетело!
gpu_freq=600
over_voltage=6
core_freq=550
temp_limit=85
```
**Результат:** загрузилась. Температура ~53°C. Стабильно.

### Этап 4: GPU 750 (повышение GPU на том же CPU)
```ini
arm_freq=2000
gpu_freq=750
over_voltage=6
core_freq=550
temp_limit=85
```
**Результат:** отличная стабильность, температура ~57°C. Браузер заметно шустрее.

### Этап 5: GPU 850 + Core 600 (потолок)
```ini
arm_freq=2000
gpu_freq=850
over_voltage=6
core_freq=600
temp_limit=85
```
**Результат:**
- GPU (V3D): 850 MHz стабильно
- Core: 425 MHz по факту (настройка не применилась)
- YouTube зависал — вероятно, видео-декодер (H264/ISP) нестабилен на 850 MHz
- USB-ошибки: `cannot get freq at ep 0x82` — core_freq=600 влиял на USB-шину
- **Система зависла**, watchdog (1 мин) перезагрузил

### Финальный этап: GPU 750 + Core 550
```ini
arm_freq=2000    ✅
gpu_freq=750     ✅
over_voltage=6   ✅
core_freq=550    ✅
temp_limit=85    ✅
```
**Результат:** стабильно, YouTube работает, браузер быстрый. Финальный конфиг.

## Детали

### Теория разгона Pi4

| Параметр | Что делает | Безопасный предел |
|----------|-----------|-------------------|
| `arm_freq` | Частота CPU (4×Cortex-A72) | до 2.0-2.15 GHz |
| `gpu_freq` | Частота GPU (V3D, H264, ISP) | до 750-800 MHz |
| `core_freq` | Частота шины SoC (L2, memory controller, USB) | до 550-600 MHz |
| `over_voltage` | Напряжение ядра (+0.025V/шаг) | 0-6 (безопасный) |

**Важно:**
- `over_voltage` выше 6 ставит перманентный бит в SoC и аннулирует гарантию
- Каждые +10°C сокращают срок жизни CPU вдвое (правило Аррениуса)
- `force_turbo=1` не рекомендуется — отключает динамическое управление частотой

### Диагностика

```bash
# Текущие частоты
vcgencmd measure_clock arm
vcgencmd measure_clock v3d
vcgencmd measure_clock core
vcgencmd measure_clock h264
vcgencmd measure_clock isp

# Температура и напряжение
vcgencmd measure_temp
vcgencmd measure_volts core
vcgencmd get_throttled

# Разбор throttled (битовая маска)
# 0x0 = всё ок
# 0x50000 = был троттлинг
# 0x80000 = было отключение питания
```

### Восстановление после неудачного разгона

1. Вынуть SD-карту из Pi4
2. Вставить в другой компьютер (или USB-кардридер)
3. Отредактировать `/boot/firmware/config.txt` — закомментировать или удалить строки разгона
4. Вставить обратно, загрузиться
5. Снизить частоту или поднять напряжение

## Ссылки

- [Raspberry Pi config.txt docs](https://www.raspberrypi.com/documentation/computers/config_txt.html)
- [Q-engineering: Overclocking Pi4](https://www.qengineering.eu/overclocking-the-raspberry-pi-4.html)
- [jerrf010/Raspberry-Pi-4-Config-Overclock (GitHub)](https://github.com/jerrf010/Raspberry-Pi-4-Config-Overclock-)
