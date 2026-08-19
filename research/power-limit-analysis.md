# CMP 90HX Power Limit Analysis

## Статус: Требуется дальнейшее исследование

Power limit 250W (вместо 320W TDP) заложен в vBIOS и защищён подписью.
Прямая модификация vBIOS невозможна без ключей NVIDIA.

## Обнаружено (2026-08-19)

### vBIOS Power Table

Найдена таблица power limits в vBIOS (94.02.74.00.05) по адресу **0x85070**:

```
Offset    Bytes           Value       Meaning
0x85070   00 00 01 00     -           Header/flags
0x85074   a0 86 01 00     100000 mW   MIN power limit (100W)
0x85078   90 d0 03 00     250000 mW   DEFAULT power limit (250W)
0x8507c   90 d0 03 00     250000 mW   MAX power limit (250W) ← TARGET
```

### Для разблокировки 320W

Изменить offset **0x8507c** с `90 d0 03 00` на `00 e2 04 00` (320000 mW = 320W).

## Методы разблокировки (по возрастанию риска)

### 1. Патч драйвера (рекомендуется)

Найти место в открытом коде драйвера, где читаются power limits из vBIOS, и патчить их на лету.

**Потенциальные точки патча:**
- GSP firmware загружает VBIOS tables в память
- RM парсит POWER_BUDGET_TABLE и кэширует лимиты
- NVML читает кэшированные лимиты через RPC

**Плюсы:** Безопасно, откатывается rmmod/modprobe
**Минусы:** Нужен реверс-инжиниринг, патч каждой новой версии драйвера

### 2. Модификация vBIOS

1. Изменить байты по адресу 0x8507c
2. Пересчитать checksum
3. Прошить через nvflash --save + nvflash vbios.rom

**Плюсы:** Постоянное решение
**Минусы:** Риск брика, нужен hardware programmer для recovery

### 3. LD_PRELOAD hook на NVML

Перехватить nvmlDeviceGetPowerManagementLimitConstraints и подменить возвращаемые значения.

**Плюсы:** Простая реализация
**Минусы:** Не влияет на реальный лимит, только на отображение

## Следующие шаги

1. Найти в открытом коде драйвера где парсится POWER_BUDGET_TABLE
2. Написать патч по аналогии с compute unlock (V67 exploit)
3. Или: получить nvflash и сделать vBIOS mod (с backup!)

## Файлы

- `/tmp/cmp90hx.rom` — оригинальный vBIOS dump (raw, 1MB)
- `~/backup_original.rom` — nvflash backup (NVGI, 999KB)
- `~/cmp90hx_320W.rom` — пропатченный raw vBIOS (не прошивается)
- Версия: 94.02.74.00.05
- Дата vBIOS: 06/07/21

## Попытки прошивки

1. **nvflash --save/flash** — работает, но требует NVGI формат
2. **nvflash -6 raw.rom** — ERROR: Invalid firmware image (raw не принимается)
3. **nvflashk** — обходит Board ID, но НЕ подпись (модифицированный vBIOS отклоняется)

## Результаты исследования (2026-08-20)

### Анализ открытого кода драйвера

Изучен исходный код NVIDIA Open GPU Kernel Modules 610.43.03:

1. **`subdeviceCtrlCmdPerfRatedTdpSetControl_KERNEL`** (`kern_perf_pwr.c`):
   - Перенаправляет вызов в GSP-RM через `NV_RM_RPC_CONTROL`
   - Логика проверки power limits в **закрытом GSP firmware**

2. **RM Control commands**:
   - `0x2080206e` — `NV2080_CTRL_CMD_PERF_RATED_TDP_GET_CONTROL`
   - `0x2080206f` — `NV2080_CTRL_CMD_PERF_RATED_TDP_SET_CONTROL`
   - `0x20800112` — `NV2080_CTRL_GPU_SET_POWER`

3. **NVML API flow**:
   ```
   nvmlDeviceSetPowerManagementLimit()
     → RM control call → GSP-RM
     → GSP проверяет limits из vBIOS
     → Возвращает NV_ERR_INVALID_ARGUMENT если > max
   ```

### Доступ к BAR0 регистрам

**Проблема:** `mmap` на `/sys/bus/pci/devices/.../resource0` возвращает EINVAL.
Причины:
- Kernel CONFIG_IO_STRICT_DEVMEM включен
- resource0 требует специальных флагов для mmap

**Альтернативы:**
- `nvidia-debugdump` — создаёт dump но не даёт прямой доступ к регистрам
- `envytools/nvapeek` — не работает когда nvidia драйвер загружен
- NVIDIA ioctl — требует правильные RM handles

### FEAT_OVR Power Register — НЕ СУЩЕСТВУЕТ

Проверены регистры в области 0x823800-0x823850:
- `0x823804` — FEAT_OVR_PLM ✅
- `0x82381C` — FEAT_OVR_SM_SPD (SS0) ✅
- `0x823820` — FEAT_OVR_SM_SPD_1 (SS1) ✅

**Power limit НЕ контролируется через FEAT_OVR!**
В отличие от SM speed (аппаратный throttle с override регистрами),
power limit — это **софтверный лимит** в GSP firmware,
который читает значения из vBIOS POWER_BUDGET_TABLE.

## Возможные пути (требуют исследования)

### 1. ~~Поиск PLM-защищённого power регистра~~ ❌ НЕ СУЩЕСТВУЕТ

### 2. Патч ioctl для подмены RM control response
Перехватить ioctl на `/dev/nvidiactl` и подменить response для
`NV2080_CTRL_CMD_PERF_RATED_TDP_GET_CONTROL` чтобы возвращал max=320W.
**Проблема:** GSP всё равно проверит при SET_CONTROL.

### 3. Hardware SPI programmer (CH341A)

### 2. Hardware SPI programmer (CH341A)
Прямая прошивка SPI flash минуя nvflash. Требует:
- CH341A programmer + SOIC8 clip
- Физический доступ к SPI chip на карте

### 3. Патч GSP firmware
Найти где GSP читает power limits из vBIOS и патчить.
Сложность сопоставима с оригинальным V67 exploit.

## Команды для трассировки

```bash
# strace nvidia-smi для захвата ioctl
strace -e ioctl -v -x nvidia-smi -pl 320 2>&1

# RATED_TDP test (не работает - NV_ERR_NOT_SUPPORTED)
# Команда 0x2080206f отклоняется GSP для этой карты
```

## LD_PRELOAD hooks

```c
// RATED_TDP hook - status=0x1f (NOT_SUPPORTED)
// Power limit constraints проверяются в GSP/NVML, не через RATED_TDP API
```
