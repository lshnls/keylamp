# Светодиодный индикатор раскладки клавиатуры на Arduino для Gnome

![KeyLamp](img/KeyLamp.gif)

Часть 1. Электронная начинка и ПО  

Необходимо:

1. Контроллер Arduino с тремя аналоговыми портами и подключением по USB. Я использовал Arduino Nano V3.0 MicroUSB Запаяна ATMEGA 328P.
2. RGB светодиод, общий анод (плюс), 5мм, 5 шт. Можно использовать светодиод с общим катодом, схема подключения светодиода при этом изменится.
3. Резисторы 220Ом 0,5вт - 3 шт.
4. Кабель Micro USB для подключения контроллера к компьютеру.
5. Компьютер с ОС Linux и Gnome. Я использовал Arch Linux и Gnome 49.

## 1. Программирование Arduino

Задаем 4 варианта подсветки, подсведка регклируется путем отправки символа в консоль tty по порту USB

Содержание скетча:

``` cpp
int pushButton = 2;

struct RGB {
  byte r;
  byte g;
  byte b;
};


// Яркость 0..255
int BRIGHT = 255;


RGB colors[] = {
  {0, 0, 0},  // черный
  {BRIGHT, 0, 0},   // красный
  {0, BRIGHT, 0},   // зелёный
  {0, 0, BRIGHT}   // синий
};

void setup() {
  pinMode(pushButton, INPUT);
  Serial.begin(9600);      // Начинаем работу с последовательным портом
  Serial.println("Start arduino...");

// тестирование при запуске
  for (int i = 3; i >=0; i--) {
    analogWrite(9,colors[i].r);
    analogWrite(10,colors[i].g);
    analogWrite(11,colors[i].b);
    delay(100);
  }
}

void loop() {
  while (Serial.available()) {       // Проверяем наличие входящих данных
    int command = Serial.read() - '0'; // Читаем команду
    
    if (0 <= command && command <= 3) {
      analogWrite(9,colors[command].r);
      analogWrite(10,colors[command].g);
      analogWrite(11,colors[command].b);
      
      Serial.print("Pressed: ");
      Serial.println(command);
    }
 //     Serial.print("Time: ");
 //     Serial.println(millis());
    delay(100);
  }
}
```


## 2. Расширение для Gnome

Яык графической оболочки рабочего стола Gnome это Javascrypt. Сама оболочка реализована как JS‑приложение, выполняющееся в среде GJS — JavaScript‑движке на базе SpiderMonkey (от Mozilla), который также используется в браузере Firefox.
Расширения для GNOME Shell также пишутся на JS.

Создадим директорию под новое расширение и перейдем а нее
- ~/.local/share/gnome-shell/extensions/ — стандартное место для расширений уровня пользователя; возможно там у вас уже что‑то есть
- input-source-monitor@eshfield — название расширения. Регламент требует две разделённые символом @ части: собственно название и подконтрольное пространство имён, например, адрес email или сайта — для наших локальных целей можно ограничиться именем пользователя.
- конструкция && cd $_ позволяет сразу же перейти в созданную директорию — параметр командной строки $_ содержит последний аргумент предыдущей команды, то есть указанный путь

``` bash
mkdir -p ~/.local/share/gnome-shell/extensions/input-source-mon@eshfield
cd $_
```
В директории расширения понадобится создать два обязательных файла:

**metadata.json**, в нем указываем минимально необходимый набор полей с основной информацией о расширении:
``` json
{
    "uuid": "input-source-monitor@eshfield",
    "name": "Keyboard Input Source Monitor",
    "description": "Monitors input source changes and notifies external script via custom D-Bus interface",
    "shell-version": ["46", "47", "48", "49"]
}
```
Здесь:
- поле идентификатора uuid должно совпадать с полным названием расширения из созданной ранее директории.
- в массиве shell-version указываются поддерживаемые версии GNOME — в нашем случае они соответствуют диапазону от Ubuntu 24.04 (GNOME 46) до Fedora 43 (GNOME 49)
Подробнее о полях этого файла можно почитать в документации: https://gjs.guide/extensions/development/creating.html

**extension.js**, здесь ожидается унаследованная от базового класса Extension реализация:
``` js
import Gio from "gi://Gio";
import GLib from "gi://GLib";
import St from "gi://St";

import { Extension } from "resource:///org/gnome/shell/extensions/extension.js";
import * as Main from "resource:///org/gnome/shell/ui/main.js";
import * as PanelMenu from "resource:///org/gnome/shell/ui/panelMenu.js";
import * as Keyboard from "resource:///org/gnome/shell/ui/status/keyboard.js";

const ICON_NAME = "format-text-plaintext-symbolic";
const ICON_STYLE = "system-status-icon";

const DBUS_INTERFACE = `
    <node>
        <interface name="org.gnome.InputSourceMonitor">
            <signal name="SourceChanged">
                <arg type="s" name="source"/>
            </signal>
        </interface>
    </node>
`;
const DBUS_NAME = "org.gnome.InputSourceMonitor";
const DBUS_PATH = "/org/gnome/InputSourceMonitor";
const DBUS_SIGNAL_NAME = "SourceChanged";
const MANAGER_SIGNAL_NAME = "current-source-changed";

export default class InputSourceMonitorExtension extends Extension {
    constructor(metadata) {
        super(metadata);

        this._dbus = null;
        this._indicator = null;
        this._manager = Keyboard.getInputSourceManager();
        this._ownerId = null;
        this._signalId = null;
    }

    enable() {
        // setup panel indicator
        this._indicator = new PanelMenu.Button(0.0, this.metadata.name, false);

        const icon = new St.Icon({
            icon_name: ICON_NAME,
            style_class: ICON_STYLE,
        });

        this._indicator.add_child(icon);
        Main.panel.addToStatusArea(this.uuid, this._indicator);

        // setup D-Bus interface
        this._dbus = Gio.DBusExportedObject.wrapJSObject(
            DBUS_INTERFACE,
            this,
        );
        this._dbus.export(Gio.DBus.session, DBUS_PATH);

        // reserve D-Bus name
        this._ownerId = Gio.DBus.session.own_name(
            DBUS_NAME,
            Gio.BusNameOwnerFlags.NONE,
            null,
            null
        );

        // subscribe to input source changes
        this._signalId = this._manager.connect(
            MANAGER_SIGNAL_NAME,
            () => {
                const source = this._manager.currentSource;
                this._dbus.emit_signal(
                    DBUS_SIGNAL_NAME,
                    GLib.Variant.new("(s)", [source.id])
                );
            },
        );
    }

    disable() {
        if (this._signalId) {
            this._manager.disconnect(this._signalId);
            this._signalId = null;
        }

        if (this._dbus) {
            this._dbus.flush();
            this._dbus.unexport();
            this._dbus = null;
        }

        if (this._ownerId) {
            Gio.DBus.session.unown_name(this._ownerId);
            this._ownerId = null;
        }

        if (this._indicator) {
            this._indicator.destroy();
            this._indicator = null;
        }
    }
}
```

Здесь:

- объявляем константы, среди которых DBUS_INTERFACE, который содержит XML‑описание структуры создаваемого интерфейса D‑Bus — имя, сигнал и аргумент:

    type="s" — строковый тип (s — string).

    name="source" — имя аргумента, значением которого будет текущая раскладка: us или ru.

- инициируем нужные переменные

- в методе enable() указываем иконку для панели — я выбрал view-wrapped-symbolic; при желании можете выбрать другую из директории /usr/share/icons/Adwaita/symbolic/status/ или заморочиться добавлением своей

- создаём и регистрируем новый интерфейс D‑Bus по описанной в константе структуре

- резервируем на D‑Bus уникальное имя, по которому внешние клиенты смогут найти и подключиться к нашему сервису

- _manager — это объект, управляющий раскладками и источниками ввода клавиатуры; он знает текущую раскладку и при помощи метода connect() даёт возможность подписаться на события её смены

- emit_signal() отправляет в созданный интерфейс сообщения, передавая из переменной source.id строковое значение текущей раскладки: us или ru

- в методе disable() очищаем ресурсы

Осталось перезапустить GNOME — выйти из сессии и зайти заново. Теперь можно активировать расширение, выполнив команду (из любой директории):

``` bash
gnome-extensions enable input-source-monitor@eshfield
```
Расширение будет автоматически запускаться со стартом системы. На верхней панели мы должны увидеть выбранную иконку:
![alt text](img/image1.png)

Можно проверить корректность работы:
``` bash
gdbus monitor --session --dest org.gnome.InputSourceMonitor
```
При смене раскладки будут видны соответствующие сообщения:

![alt text](img/image2.png)

## 3. Пишем скрипт на Python

***main.py***

``` python
import asyncio
import logging
import sys
from typing import Optional

from dbus_fast import DBusError
from dbus_fast.aio import MessageBus, ProxyInterface

import serial

START_TIMEOUT_SECONDS = 10

BUS_NAME = "org.gnome.InputSourceMonitor"
BUS_PATH = "/org/gnome/InputSourceMonitor"

RED = "1"
BLUE = "2"
COLORS = {
    "us": BLUE,
    "ru": RED,
}

PORT = "/dev/ttyUSB0"

logger = logging.getLogger("kb_indicator")

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    stream=sys.stdout,
)


async def wait_for_input_source_interface(bus: MessageBus) -> Optional[ProxyInterface]:
    for i in range(1, START_TIMEOUT_SECONDS + 1):
        try:
            introspection = await bus.introspect(BUS_NAME, BUS_PATH)
            proxy = bus.get_proxy_object(BUS_NAME, BUS_PATH, introspection)
            logger.info("Connected to D-Bus")
            return proxy.get_interface(BUS_NAME)
        except DBusError:
            logger.info(f"Try #{i}: DBus service is not ready, retrying")
            await asyncio.sleep(1)
    return None


async def main():
    
    bus: Optional[MessageBus] = None

    ser = serial.Serial(PORT, 9600)

    try:
        bus = await MessageBus().connect()

        interface = await wait_for_input_source_interface(bus)
        if interface is None:
            logger.error(f"D-Bus service is not available after {START_TIMEOUT_SECONDS} seconds")
            return

        last_source: dict[str, Optional[str]] = {"value": None}

        def handle_source_change(source: str) -> None:
            if source == last_source["value"]:
                return

            last_source["value"] = source
            color = COLORS.get(source)
            if not color:
                logger.warning(f"Unexpected input source: {source}")
                return

            # переключим индикацию
            print("Input source changed to: "+ source + ", color code: " + color)
            ser.write(color.encode())
            
        interface.on_source_changed(handle_source_change)  # type: ignore

        logger.info("Listening for input source changes. Press Ctrl+C to exit")
        await asyncio.Future()
    except DBusError as e:
        logger.error(f"D-Bus error: {e}")
    except asyncio.CancelledError:
        logger.info("Shutdown requested")
    except Exception as e:
        logger.exception(f"Unexpected error: {e}")
    finally:
    
        # погасим индикацию
        ser.write(b'0')
        ser.flush() 
        ser.close()
    
        if bus is not None:
            bus.disconnect()
        logger.info("Exit")


if __name__ == "__main__":
    asyncio.run(main())
```


Здесь:

- описаны значения цветов подсветки для раскладок: для en я выбрал нейтрально белый цвет, для ru — идеологически красный

- ожидается готовность сервиса D‑Bus и его интерфейса, чтобы подписаться на события изменения раскладки: скрипт делает несколько попыток подключения в течение отведённого времени — это защита от падения при автозапуске, когда GNOME‑сессия и связанные сервисы D‑Bus ещё не полностью инициализированы системой при старте

- при смене раскладки обработчик handle_source_change отправляет знакомую нам команду на смену цвета подсветки

- Скрипт готов, осталось позаботиться о его автоматическом запуске при входе в систему. Для этого создадим файл .desktop в нужной директории:

mkdir -p ~/.config/autostart
nano ~/.config/autostart/InputSourceMonitor.desktop

**InputSourceMonitor.desktop**

```
[Desktop Entry]
Type=Application
Name=InputMonitor
Exec=systemd-cat -t Name=InputMonitor <PATH_TO_PYTHON_BIN> <PATH_TO_MAIN_PY_FILE>
X-GNOME-Autostart-enabled=true
NoDisplay=false
Comment=Keyboard Input Monitor
```

Здесь:

- Type и Name — стандартные обязательные поля

- Exec — команда для выполнения: здесь мы будем запускать наш Python‑скрипт

- systemd-cat -t anubeam — это утилита, которая перенаправляет stdout и stderr в журнал логирования journal systemd с указанным тегом (-t anubeam), чтобы при необходимости иметь возможность посмотреть логи нашего скрипта командой journalctl --user -t anubeam -f

- <ABSOLUTE_PATH_TO_PYTHON_BIN> — абсолютный путь к исполняемому файлу Python — можно узнать командой pyenv which python, выполненной из директории со скриптом

- <ABSOLUTE_PATH_TO_MAIN_PY_FILE> — абсолютный путь к главному файлу скрипта — можно собрать из вывода команды pwd там же + /main.py

 - X-GNOME-Autostart-enabled=true — включает автозапуск в GNOME

- NoDisplay=false — так наш скрипт будет виден в настройках автозапуска, например, в приложении Tweaks (раздел Startup Applications)

Осталось ещё раз выйти из сессии GNOME и войти заново, чтобы получить готовое решение. Теперь смена раскладки будет сопровождаться соответствующей подсветкой клавиатуры, и вы всегда будете знать, какие символы набираете.
