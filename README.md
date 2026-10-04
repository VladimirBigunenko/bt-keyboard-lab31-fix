# bt-keyboard

Скрипт для Linux, который лечит классическую проблему Bluetooth-клавиатур:
система пишет «подключено», а клавиатура не печатает.

Проверено на клавиатуре **Lab31** (BLE HID, BlueZ 5.82, Parrot Security 7.4, ядро 6.17).

## Симптом

Клавиатура спарена, в интерфейсе KDE/GNOME числится подключённой, но на нажатия
не реагирует. В `/proc/bus/input/devices` её нет.

`bluetoothctl` при этом бодро сообщает:

```
Attempting to connect to FF:FF:11:B2:B1:05
Connection successful
```

А в логе BlueZ ровно одна подозрительная строка:

```
profiles/input/hog-lib.c: report_map_read_cb()
Report Map read failed: Encryption required before read/write
```

## Причина

1. BlueZ открывает BLE-соединение, ACL-линк устанавливается — отсюда честное `Connected: yes`
2. Шифрование не поднимается: ключ сопряжения (LTK) на одной из сторон протух
3. Без шифрования BlueZ не может прочитать характеристику HID Report Map
4. Не прочитав дескриптор, BlueZ **не создаёт uhid-устройство**
5. В системе нет input-устройства → ядро не генерирует события → клавиатура мертва

Типичная причина порчи ключа — перезагрузка прошивки Bluetooth-контроллера.
В `dmesg` это выглядит так:

```
Bluetooth: hci0: Found device firmware: intel/ibt-11-5.sfi
Bluetooth: hci0: Firmware loaded in 1897384 usecs
```

**Важно:** дело не в отсутствии драйвера в ядре. Проверялось экспериментально —
создание `uhid`-устройства с шиной `BUS_BLUETOOTH` показало, что `hid-generic`
(алиас `hid:b*g*v*p*`, wildcard по всем шинам) прекрасно связывает такие
устройства и создаёт `/dev/input/event*`. Проблема целиком на стороне сопряжения.

## Установка

```sh
curl -fsSL https://raw.githubusercontent.com/VladimirBigunenko/bt-keyboard-lab31-fix/main/bt-keyboard \
     -o ~/.local/bin/bt-keyboard
chmod +x ~/.local/bin/bt-keyboard
```

Убедись, что `~/.local/bin` есть в `PATH`.

## Использование

```
bt-keyboard                     # проверить, подключить, при неудаче перепарить
bt-keyboard --repair            # сразу перепарить (нужен режим сопряжения)
bt-keyboard --status            # только показать состояние
bt-keyboard --test              # после настройки проверить живыми нажатиями
bt-keyboard --no-test           # не предлагать проверку нажатий
bt-keyboard --name "Lab31"      # другое имя устройства
bt-keyboard --addr XX:XX:...    # задать адрес вручную
```

Обычный запуск делает всё сам:

```
$ bt-keyboard
==> Проверяю Bluetooth-адаптер
  ✓ адаптер 28:16:AD:28:4A:BE работает
  ✓ клавиатура «Lab31» уже работает

  ✓ клавиатура «Lab31» зарегистрирована как input-устройство (/dev/input/event18)
    BlueZ:
      Name: Lab31 - Keyboard
      Paired: yes
      Bonded: yes
      Trusted: yes
      Connected: yes
? Проверить клавиатуру живыми нажатиями [y/N] y

==> Проверяю живыми нажатиями (/dev/input/event18)
    нажми клавишу на «Lab31» — если она спала, первое нажатие её разбудит
    жду первого нажатия до 20 с, дальше проверка завершится сама
    [     НАЖАТА] ENTER
    [   отпущена] ENTER
    [     НАЖАТА] TAB
    [   отпущена] TAB
    событий поймано: 4
  ✓ нажатия доходят — клавиатура работает
```

### Перепаровка

Для `--repair` клавиатуру нужно перевести в **режим сопряжения** — индикатор
Bluetooth должен быстро мигать. Обычно это `Fn + 1/2/3`, `Fn + Q/W/E` или
отдельная кнопка `BT`, удерживаемая 3–5 секунд. И клавиатура не должна быть
подключена к телефону или другому компьютеру.

Скрипт сам сканирует эфир до 90 секунд и пишет, что происходит.

## Что учтено

Это не просто обёртка вокруг `bluetoothctl remove && bluetoothctl pair`.
Грабли, на которые легко наступить руками, заложены в логику:

| Грабли | Решение |
|---|---|
| Активное сканирование ломает сопряжение | Скан гарантированно останавливается **до** `pair` |
| `Device not available` при паринге | Адрес берётся из аргумента или списка известных, но паринг идёт строго после обнаружения в эфире |
| BLE-адрес мог смениться | Если известного адреса нет в эфире, устройство ищется **по имени**, адрес обновляется |
| Медленный режим сопряжения | До 9 заходов по 10 секунд (~90 с) с подсказками |
| «Connected: yes», а клавиатура мертва | Проверка идёт по появлению **input-устройства** в `/proc/bus/input/devices`, а не по статусу BlueZ |
| Первое подключение не всегда поднимает HID | До 3 попыток с ожиданием до 20 секунд |
| Адаптер Bluetooth выключен | Проверяется и включается автоматически |
| Клавиатура спит | Проверка нажатий ждёт **первое** нажатие (до 20 с), затем завершается сама через 6 секунд тишины |

Скрипт не требует root для настройки: всё идёт через BlueZ по D-Bus от
пользователя. Root нужен только для проверки нажатий (чтение `/dev/input/event*`).

## Требования

- Linux с BlueZ (`bluetoothctl`) версии 5.55+ — нужен флаг `--timeout`
- `python3` — только для проверки нажатий
- `sudo` или `pkexec` — только для проверки нажатий

Установка на Debian/Ubuntu/Parrot:

```sh
sudo apt install bluez python3
```

## Диагностика вручную

Если хочется проверить самостоятельно, что дело именно в этом:

```sh
# 1. Есть ли input-устройство клавиатуры?
grep -i имя_клавиатуры /proc/bus/input/devices

# 2. Что говорит ядро о Bluetooth-HID?
sudo dmesg | grep -i hid

# 3. Что говорит BlueZ?
sudo journalctl -u bluetooth | grep -iE 'hog|encrypt|report map'
```

Строка `Report Map read failed: Encryption required before read/write`
означает ровно то, что лечит этот скрипт.

## Ограничения

- Скрипт чинит **сопряжение**, а не драйверы. Если в ядре действительно нет
  поддержки Bluetooth-HID, он не поможет — но такие ядра в современных
  дистрибутивах не встречаются.
- Если клавиатура физически неисправна или разряжена, скрипт этого не исправит.
- Проверка нажатий ловит события `EV_KEY`; на экзотических раскладках имена
  клавиш могут отображаться кодами вида `KEY_123`.

## Лицензия

MIT — см. [LICENSE](LICENSE).

---

## English

`bt-keyboard` is a Linux script that fixes the classic Bluetooth keyboard
problem where the device shows as *connected* but never types.

**Root cause:** a stale pairing key (LTK). BlueZ establishes the BLE link
(`Connection successful`) but cannot raise encryption, so it fails to read the
HID Report Map (`Report Map read failed: Encryption required before read/write`).
Without the descriptor BlueZ never creates a `uhid` device, so no input device
exists and the kernel generates no events.

**Fix:** remove the stale bond and re-pair. The script automates this, including
the pitfalls — scanning must stop before pairing, the BLE address may have
changed, and success is verified by an actual input device appearing in
`/proc/bus/input/devices` rather than by BlueZ's `Connected: yes`.

Tested with a Lab31 BLE keyboard. Messages are in Russian.
