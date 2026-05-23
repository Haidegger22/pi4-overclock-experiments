# Pi4 Overclock Experiments

Эксперименты с разгоном Raspberry Pi 4 (BCM2711, rev 1.5) — параметры, результаты, стабильность.

## Стенд

- **Плата:** Raspberry Pi 4 Model B Rev 1.5
- **ОС:** Debian 13 (trixie), Raspberry Pi OS kernel 6.18.29
- **Охлаждение:** радиатор + 2 вентилятора
- **Блок питания:** UPS 5V 3A (2x18650)
- **Нагрузка:** labwc (Wayland), wf-panel-pi, браузер (вкладки, YouTube)
- **Дисплеи:** DSI 4.3" (800x480) + HDMI (1360x768)

## Сводка настроек

| Компонент | Stock | Итоговый разгон | Прирост |
|-----------|------|----------------|---------|
| **CPU** (arm_freq) | 1.8 GHz | **2.0 GHz** | +11% |
| **GPU** (gpu_freq) | 500 MHz | **700 MHz** | +40% |
| **Core** (core_freq) | 500 MHz | **550 MHz** | +10% |
| **Напряжение** (over_voltage) | 0 | **6** | безопасно |
| **Температура макс** (100% CPU+GPU) | ~50C | **~65C** | отлично |
| **Троттлинг** | 0 | **0** | ни разу |

## config.txt (итоговый рабочий)

```ini
arm_freq=2000
gpu_freq=700
over_voltage=6
core_freq=550
temp_limit=85
```

## Хронология экспериментов

### Этап 1: CPU 2.0 GHz + GPU 600 + over_voltage=2
```ini
arm_freq=2000    -- система не загрузилась :(
gpu_freq=600
over_voltage=2
core_freq=550
```
**Результат:** не загрузилась. SD-карта вынута, config.txt почищен на Pi5.

### Этап 2: CPU 1.9 GHz + GPU 550 + over_voltage=4
```ini
arm_freq=1900    -- стабильно
gpu_freq=550
over_voltage=4
core_freq=525
temp_limit=85
```
**Результат:** загрузилась. Температура ~50C. Работает.

### Этап 3: CPU 2.0 GHz + GPU 600 + over_voltage=6
```ini
arm_freq=2000    -- взлетело!
gpu_freq=600
over_voltage=6
core_freq=550
temp_limit=85
```
**Результат:** загрузилась. Температура ~53C. Стабильно.

### Этап 4: GPU 750 (повышение GPU)
```ini
arm_freq=2000
gpu_freq=750
over_voltage=6
core_freq=550
temp_limit=85
```
**Результат:** загрузилась. Температура ~57C. Браузер шустрее.
**Спустя час:** GPU (V3D) начал виснуть (ошибка 0x00001000, Resetting GPU for hang). 750 MHz нестабилен.

### Этап 5: GPU 850 + Core 600 (потолок)
```ini
arm_freq=2000
gpu_freq=850
over_voltage=6
core_freq=600
temp_limit=85
```
**Результат:**
- GPU (V3D): 850 MHz стабильно в простое, но декодер нестабилен
- Core: 425 MHz по факту (настройка не применилась)
- YouTube зависал
- USB-ошибки: cannot get freq at ep 0x82
- **Система зависла**, watchdog (1 мин) перезагрузил

### Этап 6: GPU 700 (стабильный финал)
```ini
arm_freq=2000    -- стабильно
gpu_freq=700     -- стабильно
over_voltage=6   -- безопасно
core_freq=550    -- стабильно
temp_limit=85    -- запас
```
**Результат:**
- V3D hang счётчик: 0
- Температура: ~60C под нагрузкой
- YouTube работает, браузер быстрый
- **Стабильно. Финальный конфиг.**

## Почему 750 MHz не взлетел

На этом экземпляре Pi4 (rev 1.5) GPU V3D на 750 MHz стабилен в простое, но под реальной нагрузкой (WebGL, браузер) через 30-60 минут начинает виснуть каждую секунду:

```
v3d fec00000.v3d: *ERROR* V3D_ERR_STAT: 0x00001000
v3d fec00000.v3d: *ERROR* Resetting GPU for hang.
```

Драйвер сбрасывает GPU, но цикл повторяется. 700 MHz — стабильный потолок для этого экземпляра.

## Детали

### Параметры config.txt

| Параметр | Что делает | Безопасный предел |
|----------|-----------|-------------------|
| arm_freq | Частота CPU (4x Cortex-A72) | до 2.0-2.15 GHz |
| gpu_freq | Частота GPU (V3D, H264, ISP) | до 700-750 MHz |
| core_freq | Частота шины SoC (L2, USB) | до 550-600 MHz |
| over_voltage | Напряжение ядра (+0.025V/шаг) | 0-6 (безопасный) |

**Важно:**
- over_voltage выше 6 ставит перманентный бит в SoC (аннулирует гарантию)
- Каждые +10C сокращают срок жизни CPU вдвое
- force_turbo=1 не рекомендуется (отключает динамику)

### Диагностика

```bash
# Текущие частоты
vcgencmd measure_clock arm
vcgencmd measure_clock v3d
vcgencmd measure_clock core

# Температура и напряжение
vcgencmd measure_temp
vcgencmd measure_volts core
vcgencmd get_throttled

# Проверка GPU hang (если >0 -- GPU нестабилен)
dmesg | grep -c 'Resetting GPU for hang'
```

### Восстановление после неудачного разгона

1. Вынуть SD-карту
2. Вставить в другой компьютер / кардридер
3. Отредактировать /boot/firmware/config.txt — убрать/закомментировать строки разгона
4. Вставить обратно, загрузиться

## Ссылки

- Raspberry Pi config.txt docs: https://www.raspberrypi.com/documentation/computers/config_txt.html
- Q-engineering: https://www.qengineering.eu/overclocking-the-raspberry-pi-4.html
- jerrf010 config: https://github.com/jerrf010/Raspberry-Pi-4-Config-Overclock-
