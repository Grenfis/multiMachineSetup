# Автопереключение монитора между Mac и Windows по USB-переключателю

Один монитор, две машины (Mac и Windows 11) и **одна общая клавиатура**. Клавиатура физически перекидывается между компьютерами USB-переключателем. Система устроена так, что монитор **автоматически показывает тот компьютер, к которому сейчас подключена клавиатура** — переключать вход монитора руками не нужно.

---

## 1. Как это работает

### Идея

- Mac подключён к монитору по **HDMI** (вход `17`).
- Windows подключён к монитору по **DisplayPort** (вход `15`).
- Клавиатура одна, ходит между машинами через USB-переключатель.
- На каждой машине есть «сторож», который ловит момент, когда **клавиатуру у него забрали**, и переключает монитор на вход *другой* машины.

Оба сторожа событийные, опроса нет: на Mac — постоянный USB-watcher Hammerspoon, на Windows — задача планировщика, которая просыпается по событию отключения USB-устройства, отрабатывает пару секунд и завершается.

### Схема срабатываний

| Событие | Кто реагирует | Что делает | Итог |
|---|---|---|---|
| Клаву переключили **на Windows** (ушла с Мака) | Mac | вход монитора → **DisplayPort (15)** | Виден Windows |
| Клаву переключили **на Mac** (ушла с Windows) | Windows | вход монитора → **HDMI (17)** | Виден Mac |

### Почему именно так (важное ограничение)

Переключение входа монитора идёт по протоколу **DDC/CI** — команда передаётся по видеокабелю. У этого монитора (и у многих других) есть ограничение: **компьютер может переключить монитор ТОЛЬКО «от себя», пока его вход активен**. Вернуть монитор к своему входу, когда он неактивен, — нельзя (команда не доходит/игнорируется).

Поэтому мы не пытаемся «показать себя», а делаем наоборот: каждая машина, **теряя клавиатуру, уводит монитор на соседа**. В этот момент она ещё активна на экране, её DDC-команда проходит. Получается надёжная двусторонняя схема без конфликтов.

---

## 2. Что нужно (оборудование)

- **USB-переключатель** для клавиатуры между двумя ПК — например **Ugreen CM662**. Клавиатура (или её приёмник) втыкается в свитч, свитч кнопкой перекидывает её между Mac и Windows.
- Монитор с поддержкой **DDC/CI** (в OSD-меню должно быть включено «DDC/CI»).
- Mac ↔ монитор по **HDMI**, Windows ↔ монитор по **DisplayPort** (либо наоборот — тогда поменяйте коды входов местами, см. ниже).

> **Коды входов монитора (VCP 0x60):** `15` = DisplayPort 1, `17` = HDMI 1. Если у вас другие разъёмы (DP2, HDMI2 и т.п.) — коды могут отличаться; проверьте в утилите (шаги ниже).

---

## 3. Настройка Mac

### 3.1. Установить ПО

```bash
# Lunar — управление монитором по DDC (CLI)
# Скачать с https://lunar.fyi  и один раз запустить приложение.

# Hammerspoon — «сторож» USB-событий
brew install --cask hammerspoon
```

Запустить Hammerspoon и выдать ему доступ:
`Системные настройки → Приватность и безопасность → Универсальный доступ (Accessibility)` → включить **Hammerspoon**.

Проверить, что CLI `lunar` на месте (путь используется в конфиге):
```bash
which lunar     # ожидаем ~/.local/bin/lunar
```

### 3.2. Узнать имя клавиатуры (как её видит Mac)

```bash
system_profiler SPUSBDataType | grep -i "Product Name"
```
Найдите строку своей клавиатуры (в этом сетапе — `ARDOR Wakizashi`). Достаточно уникальной подстроки имени (`ARDOR`).

### 3.3. Конфиг Hammerspoon

Создать файл `~/.hammerspoon/init.lua`:

```lua
-- Клавиатура ARDOR Wakizashi отключена от Мака -> монитор на DisplayPort (Windows).
-- Подключение НЕ обрабатываем: возврат монитора на HDMI делает Windows-машина.
-- Метод: сырой DDC (VCP 0x60), т.к. `lunar set input` этот монитор для HDMI игнорирует.

require("hs.ipc") -- включает командную строку `hs`

local LUNAR = "/Users/{username}/.local/bin/lunar"
local KEYBOARD_MATCH = "ARDOR"          -- подстрока в productName клавиатуры

local function setInput(value)
    -- Монитор P27FBB игнорирует `lunar set input` для HDMI, поэтому шлём сырой DDC (VCP 0x60)
    local cmd = LUNAR .. " ddc first 0x60 " .. value
    local out, ok = hs.execute(cmd, true) -- true = загрузить окружение пользовательского shell
    hs.printf("[kbd-input] %s -> ok=%s out=%s", cmd, tostring(ok), (out or ""):gsub("%s+$", ""))
    hs.notify.new({
        title = "Lunar input",
        informativeText = "input " .. value .. (ok and " ✓" or " ✗"),
        withdrawAfter = 3,
    }):send()
end

-- Только ОТКЛЮЧЕНИЕ клавиатуры от Мака: Mac (сейчас активен на HDMI) переключает
-- монитор на DisplayPort (Windows). Возврат на HDMI НЕ делаем — им занимается сама
-- Windows-машина при потере клавиатуры (монитор не даёт вернуть себя к неактивному входу).
usbWatcher = hs.usb.watcher.new(function(event)
    local name = event.productName or ""
    if name:find(KEYBOARD_MATCH) and event.eventType == "removed" then
        setInput(15) -- DisplayPort
    end
end)

usbWatcher:start()
hs.printf("[kbd-input] watcher started, matching '%s'", KEYBOARD_MATCH)
```

Под себя поменяйте:
- `LUNAR` — путь к вашему `lunar` (см. `which lunar`);
- `KEYBOARD_MATCH` — подстрока имени вашей клавиатуры;
- `15` — код входа, куда уводить (вход Windows-машины).

### 3.4. Применить и включить автозапуск

- В меню-баре Hammerspoon → **Reload Config** (или в терминале `hs -c "hs.reload()"`).
- В настройках Hammerspoon включить **Launch Hammerspoon at login** (или из терминала: `hs -c "hs.autoLaunch(true)"`).

### 3.5. Проверка (Mac)

Убрать клавиатуру с Мака (переключить свитчем на Windows) → монитор должен уйти на DisplayPort, всплывёт уведомление `Lunar input 15 ✓`. Лог событий — в **Hammerspoon Console** (`[kbd-input] ...`).

---

## 4. Настройка Windows 11

### 4.1. Установить ПО

- **ControlMyMonitor** (NirSoft, бесплатная, портативная): https://www.nirsoft.net/utils/control_my_monitor.html
  Распаковать в `C:\Tools\ControlMyMonitor\`.

### 4.2. Узнать ID монитора и код входа

1. Запустить `ControlMyMonitor.exe`, найти свой монитор.
2. Скопировать **Monitor ID** (вида `\\.\DISPLAY1\Monitor0`) — либо использовать `Primary`.
3. Найти VCP-код **`60` (Input Select)** — текущее значение будет `15` (DisplayPort, т.к. Windows сейчас активен).

### 4.3. Узнать VID/PID клавиатуры (для точного матча)

Проще всего — со стороны Мака:
```bash
ioreg -rc IOUSBHostDevice -l -w0 | grep -iA6 "ARDOR" | grep -iE "idVendor|idProduct"
```
Перевести из десятичного в hex. В этом сетапе клавиатура ARDOR Wakizashi = **`VID_0C45&PID_8006`**.

Либо на Windows: `Диспетчер устройств → клавиатура → Свойства → Сведения → ИД оборудования` — там будет `VID_xxxx&PID_xxxx`.

### 4.4. Проверить переключение вручную

⚠️ Экран уйдёт на Mac — вернуть кнопками монитора или клавиатурой на Маке:
```
"C:\Tools\ControlMyMonitor\ControlMyMonitor.exe" /SetValue "Primary" 60 17
```
`60` = VCP 0x60 (вход), `17` = HDMI 1. Если монитор ушёл на HDMI — метод рабочий.

### 4.5. Скрипт-сторож

Создать `C:\Tools\switch-on-kbd-remove.ps1`:

```powershell
# Переключить монитор на HDMI (Mac), когда клавиатура ARDOR уходит с этой Windows-машины.
# Запускается задачей MonitorSwitchOnKbdRemove по событию Kernel-PnP 1010/1011 (устройство удалено).
param([string]$DeviceId = "")

$cmm     = "C:\Tools\ControlMyMonitor\ControlMyMonitor.exe"
$monitor = "Primary"              # или конкретный ID, напр. "\\.\DISPLAY1\Monitor0"
$vcp     = 60                     # 0x60 = Input Select
$hdmi    = 17                     # HDMI 1
$hwid    = "VID_0C45&PID_8006"    # ARDOR Wakizashi

function Kbd-Present {
    [bool](Get-PnpDevice -PresentOnly -ErrorAction SilentlyContinue |
        Where-Object { $_.InstanceId -like "*$hwid*" })
}

# При ручном запуске задачи подстановка из события не выполняется и сюда приходит
# нераскрытый литерал "$(DeviceId)" — считаем его отсутствующим значением.
if ($DeviceId -like '$(*') { $DeviceId = "" }

# Клава и мышь висят на USB-свитче: при нажатии кнопки событие приходит на свитч, а не на
# клавиатуру. Поэтому по устройству не фильтруем — отсекаем только заведомо чужие шины
# (аудио, WiFi-Direct, RAS), а факт ухода клавиатуры проверяем ниже.
if ($DeviceId -and $DeviceId -notlike "USB\*") { exit 0 }

Start-Sleep -Milliseconds 1500    # дать PnP удалить дочерние узлы свитча
if (Kbd-Present) { exit 0 }

& $cmm /SetValue $monitor $vcp $hdmi
```

Под себя: `$hwid` — VID/PID вашей клавиатуры; `$hdmi` — код входа Мака.

> **Почему скрипт не фильтрует событие по VID/PID клавиатуры.** Это главная ловушка всей схемы. Клавиатура воткнута в USB-свитч, и при нажатии кнопки Windows пишет в журнал событие удаления **самого свитча**, а не клавиатуры — клавиатура исчезает молча, как его дочерний узел. В этом сетапе за 51 день: событий по свитчу — **698**, по клавиатуре — **8** (это редкие случаи, когда её отсоединяли напрямую). Если отфильтровать триггер по `VID_0C45`, система почти никогда не сработает.
>
> Поэтому триггер ловит **все** события удаления, а решение принимает скрипт — по факту отсутствия клавиатуры. Отсюда же:
> - фильтр по подстроке в самом триггере невозможен в принципе: XPath журнала Windows **не поддерживает `contains()`**, только точное сравнение;
> - точный фильтр по `DeviceInstanceId` тоже ненадёжен — путь меняется при перетыкании устройства в другой USB-порт;
> - пауза 1.5 с нужна, потому что дочерние узлы свитча удаляются из PnP не мгновенно.

### 4.6. Запуск по событию отключения устройства (Планировщик заданий)

Событие берём из журнала **`Microsoft-Windows-Kernel-PnP/Device Management`**, код **1010** («устройство было неожиданно удалено»; заодно ловим 1011 — то же самое по смыслу). Этот журнал включён в Windows 11 по умолчанию.

В обоих вариантах ниже задача должна быть с галками **«Выполнять с наивысшими правами»** и **«Выполнять только для вошедших пользователей»**. Второе обязательно: DDC/CI требует доступа к интерактивному рабочему столу, из сессии 0 под `SYSTEM` ControlMyMonitor монитор не увидит.

#### Вариант А — импорт XML (рекомендуется)

Передаёт в скрипт ID удалённого устройства (`ValueQueries`), за счёт чего заведомо чужие события (аудио, WiFi-Direct, RAS) отсекаются мгновенно, без обращения к PnP. Через GUI так настроить нельзя.

Сохранить в `C:\Tools\MonitorSwitchOnKbdRemove.xml`, заменив `{КОМПЬЮТЕР}\{username}` на своего пользователя (`whoami` покажет):

```xml
<?xml version="1.0" encoding="UTF-16"?>
<Task version="1.4" xmlns="http://schemas.microsoft.com/windows/2004/02/mit/task">
  <RegistrationInfo>
    <Author>{КОМПЬЮТЕР}\{username}</Author>
    <Description>Переключает вход монитора на HDMI (Mac), когда клавиатура уходит с этой машины (нажата кнопка USB-свитча). Триггер: Kernel-PnP 1010/1011.</Description>
    <URI>\MonitorSwitchOnKbdRemove</URI>
  </RegistrationInfo>
  <Principals>
    <Principal id="Author">
      <UserId>{КОМПЬЮТЕР}\{username}</UserId>
      <LogonType>InteractiveToken</LogonType>
      <RunLevel>HighestAvailable</RunLevel>
    </Principal>
  </Principals>
  <Settings>
    <DisallowStartIfOnBatteries>false</DisallowStartIfOnBatteries>
    <StopIfGoingOnBatteries>false</StopIfGoingOnBatteries>
    <ExecutionTimeLimit>PT1M</ExecutionTimeLimit>
    <MultipleInstancesPolicy>IgnoreNew</MultipleInstancesPolicy>
    <IdleSettings>
      <StopOnIdleEnd>false</StopOnIdleEnd>
      <RestartOnIdle>false</RestartOnIdle>
    </IdleSettings>
    <UseUnifiedSchedulingEngine>true</UseUnifiedSchedulingEngine>
  </Settings>
  <Triggers>
    <EventTrigger>
      <Subscription>&lt;QueryList&gt;&lt;Query Id="0" Path="Microsoft-Windows-Kernel-PnP/Device Management"&gt;&lt;Select Path="Microsoft-Windows-Kernel-PnP/Device Management"&gt;*[System[(EventID=1010 or EventID=1011)]]&lt;/Select&gt;&lt;/Query&gt;&lt;/QueryList&gt;</Subscription>
      <ValueQueries>
        <Value name="DeviceId">Event/EventData/Data[@Name="DeviceInstanceId"]</Value>
      </ValueQueries>
    </EventTrigger>
  </Triggers>
  <Actions Context="Author">
    <Exec>
      <Command>powershell.exe</Command>
      <Arguments>-NoProfile -WindowStyle Hidden -ExecutionPolicy Bypass -File "C:\Tools\switch-on-kbd-remove.ps1" -DeviceId "$(DeviceId)"</Arguments>
    </Exec>
  </Actions>
</Task>
```

Зарегистрировать в PowerShell **от администратора**:

```powershell
Register-ScheduledTask -TaskName 'MonitorSwitchOnKbdRemove' `
  -Xml (Get-Content 'C:\Tools\MonitorSwitchOnKbdRemove.xml' -Raw) -Force
```

> То же самое можно сделать мышкой: **Планировщик заданий → Импортировать задачу…** → выбрать этот XML.

#### Вариант Б — через GUI, без передачи ID устройства

Проще, но `ValueQueries` в интерфейсе не задаются, поэтому скрипт запускается **без** `-DeviceId` и каждый раз делает полную проверку PnP. Работает точно так же, просто чуть чаще срабатывает вхолостую.

1. **Планировщик заданий** → **Создать задачу** (не «простую»).
2. **Общие:** имя `MonitorSwitchOnKbdRemove`; «Выполнять с наивысшими правами»; «Выполнять только для вошедших пользователей».
3. **Триггеры** → Создать → Начать задачу: **«При событии»**, режим «Базовый»:
   - Журнал: `Microsoft-Windows-Kernel-PnP/Device Management`
   - Источник: `Kernel-PnP`
   - Код события: `1010`

   (базовый режим принимает только один код — 1010 достаточно, 1011 встречается редко).
4. **Действия** → Создать → Программа `powershell.exe`, аргументы:
   ```
   -NoProfile -WindowStyle Hidden -ExecutionPolicy Bypass -File "C:\Tools\switch-on-kbd-remove.ps1"
   ```
5. **Условия:** снять галку «Запускать только при питании от сети» (для ноутбука).
6. **Параметры:** «Остановить задачу, выполняемую дольше» → 1 час можно уменьшить; при повторном срабатывании — «Не запускать новый экземпляр».

### 4.7. Проверка (Windows)

1. Задача на месте и **не висит** в памяти — состояние должно быть `Ready`, а не `Running`:
   ```powershell
   Get-ScheduledTask -TaskName MonitorSwitchOnKbdRemove | Select-Object State
   ```
2. Триггер срабатывает — проверяется без трогания свитча: вынуть любую USB-флешку и посмотреть,
   что задача запускалась (монитор при этом переключиться не должен — клавиатура на месте):
   ```powershell
   Get-ScheduledTaskInfo -TaskName MonitorSwitchOnKbdRemove | Select-Object LastRunTime, LastTaskResult
   ```
   `LastRunTime` обновился, `LastTaskResult` = 0.
3. Сквозной тест: нажать кнопку свитча (клавиатура уходит на Mac) → монитор уходит на HDMI за ~2–3 с. Стоит прогнать 2–3 раза туда-обратно.

> Пункт «правой кнопкой → **Выполнить**» для этой задачи не показателен: при ручном запуске подстановка `$(DeviceId)` не выполняется, а клавиатура в этот момент на месте — скрипт корректно завершится, ничего не переключив.

---

## 5. Итоговый сценарий использования

1. Нажали кнопку USB-переключателя (Ugreen CM662) → клавиатура ушла на другую машину.
2. Машина, у которой клаву забрали, замечает это и переводит монитор на вход второй машины.
3. На экране — тот компьютер, за которым вы сейчас печатаете. Руками монитор не трогаем.

---

## 6. Нагрузка на систему

- **Mac (Hammerspoon):** полностью событийный, в простое ~0% CPU, ~30–50 МБ RAM.
- **Windows (задача планировщика):** в простое **ничего не запущено** — ни процесса, ни памяти. Задача просыпается только при отключении USB-устройства: в этом сетапе ~30 раз в сутки, каждый раз на доли секунды (события, не связанные с USB, отсекаются за ~100 мс). Для сравнения, прежняя версия с опросом раз в 2 сек делала ~43 000 проверок PnP в сутки и постоянно держала `powershell.exe` в памяти.
- На игры не влияет, выгружать перед игрой не нужно.

---

## 7. Диагностика

- **Ничего не переключается на Маке** — проверьте: Hammerspoon запущен, конфиг перезагружен, имя в `KEYBOARD_MATCH` совпадает; смотрите Hammerspoon Console.
- **`lunar set input` не переключает на HDMI** — это ожидаемо для таких панелей, используем сырой `lunar ddc first 0x60 <код>` (уже в конфиге).
- **На Windows не переключается** — проверьте, что `ControlMyMonitor /SetValue ... 60 17` работает вручную; что `VID_xxxx&PID_xxxx` в скрипте верный; при упрямой панели пробуйте альтернативы **winddcutil** или **Monitorian**.
- **Скрипт не реагирует на отключение (Windows)** — проверьте `$hwid`: в PowerShell выполните `Get-PnpDevice -PresentOnly | Where-Object InstanceId -like "*VID_0C45*"` при подключённой клавиатуре; строка должна находиться. Если нет — уточните VID/PID клавиатуры и подставьте в `$hwid`.
- **Задача создана, но ни разу не запускалась** (`LastTaskResult` = 267011, `LastRunTime` = 30.11.1999) — значит подходящего события не было. Убедитесь, что событие вообще пишется: нажмите кнопку свитча и посмотрите свежие записи:
  ```powershell
  Get-WinEvent -LogName 'Microsoft-Windows-Kernel-PnP/Device Management' -MaxEvents 20 |
    Where-Object Id -eq 1010 |
    Select-Object TimeCreated, @{n='Dev';e={$_.Properties[0].Value}}
  ```
  Если записей нет совсем — проверьте, что журнал включён (Просмотр событий → Журналы приложений и служб → Microsoft → Windows → Kernel-PnP → Device Management).
- **В журнале есть события свитча, но нет событий клавиатуры** — это нормально и так и задумано, см. врезку в п. 4.5. Не пытайтесь сузить триггер до VID/PID клавиатуры.
- **Переключается не всегда, через раз** — клавиатура не успевает пропасть из PnP к моменту проверки. Увеличьте паузу в скрипте: `Start-Sleep -Milliseconds 1500` → `2500`–`3000`.
- **Монитор уходит на Mac, когда компьютер засыпает** — ожидаемый побочный эффект: при засыпании USB-устройства отключаются, и сторож видит это как уход клавиатуры. Поведение то же, что и у прежней версии с опросом.
- **Не тот вход** — уточните коды VCP 0x60 для ваших разъёмов (Lunar: `lunar ddc first 0x60 read`; ControlMyMonitor: значение поля `60`) и поставьте нужные числа.
- **Монитор вообще не слушает DDC** — включите **DDC/CI** в OSD-меню монитора.

---

## 8. Справочник кодов входов (VCP 0x60, стандарт)

| Код | Вход |
|----:|------|
| 15  | DisplayPort 1 |
| 16  | DisplayPort 2 |
| 17  | HDMI 1 |
| 18  | HDMI 2 |
| 3   | DVI 1 |
| 27  | USB-C (у части мониторов) |

> На нестандартных панелях значения могут отличаться — сверяйтесь с реальным чтением VCP 0x60.
