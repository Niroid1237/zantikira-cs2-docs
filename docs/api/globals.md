# globals

## z.api

```lua
z.api.version  -- "0.7.0"
z.api.game     -- "cs2"
z.api.script   -- абсолютный путь текущего скрипта
z.api.ffi      -- true, если FFI включён
```

`z.api.ffi` — снимок флага на момент создания state. После переключения
чекбокса скрипт перезагружается, поэтому значение всегда соответствует
библиотекам, которые реально открыты.

## z.events

```lua
z.events.create_move
z.events.paint
z.events.ui
z.events.shutdown
```

Каждое событие — таблица с методом `:add(fn)`. Подробности:
[события](events.md).

## Модули API

| Module | Description |
|---|---|
| `z.entitylist` | снимки игроков и сущностей, кости, хитбоксы, видимость, история |
| `z.combat` | autowall, backtrack-записи |
| `z.trace` | луч и hull |
| `z.engine` | состояние движка, client commands, prediction |
| `z.render` | примитивы, текст, изображения, draw-list |
| `z.assets` | загрузка изображений, SVG и шрифтов |
| `z.net` | HTTP, DNS, TCP, UDP |
| `z.settings` | чтение и запись настроек чита |
| `z.scripts` | перезагрузка и папка скриптов |
| `z.ui` | окна и виджеты |
| `z.math` | векторы, углы, FOV |
| `z.globals` | состояние кадра |
| `z.time`, `z.buttons`, `z.log`, `print` | время, кнопки, логи |

## z.globals

```lua
local curtime = z.globals.curtime()
local tick = z.globals.tick_count()
local in_game = z.globals.is_in_game()
local dt = z.globals.frame_time()
local fps = z.globals.fps()
local all = z.globals.snapshot()
```

| Function | Return | Description |
|---|---|---|
| `z.globals.curtime()` | number или nil | текущее игровое время |
| `z.globals.tick_count()` | integer или nil | номер тика |
| `z.globals.is_in_game()` | boolean | есть ли локальный pawn/observer |
| `z.globals.frame_time()` | number | delta кадра |
| `z.globals.fps()` | number | FPS xdraw |
| `z.globals.snapshot()` | table | все значения одной таблицей |

`curtime` и `tick_count` возвращают `nil`, пока `global_vars` недоступны
(например, до входа в игру). `snapshot()` просто не добавляет эти поля, если
значений нет.

## z.time

| Function | Return | Description |
|---|---|---|
| `z.time.now()` | number | Unix time в секундах |
| `z.time.tick()` | number | секунды с запуска Windows |
| `z.time.frame()` | number | delta последнего кадра |

`now()` — float с точностью 100 ns, приведённый к Unix epoch. `tick()` —
`GetTickCount64() / 1000`. `frame()` — значение xdraw для текущего кадра.

## z.buttons

| Field | Value | Bit |
|---|---:|---:|
| `attack` | 1 | 0 |
| `jump` | 2 | 1 |
| `duck` | 4 | 2 |
| `forward` | 8 | 3 |
| `back` | 16 | 4 |
| `use` | 32 | 5 |
| `left` | 128 | 7 |
| `right` | 256 | 8 |
| `move_left` | 512 | 9 |
| `move_right` | 1024 | 10 |
| `second_attack` | 2048 | 11 |
| `reload` | 8192 | 13 |
| `sprint` | 65536 | 16 |
| `scoreboard` | 8589934592 | 33 |
| `zoom` | 17179869184 | 34 |
| `look_at_weapon` | 34359738368 | 35 |

Это bitmask, а не отдельные кнопки. Используй `cmd:set_button(mask, pressed)`,
чтобы не затереть остальные клавиши.

## z.log и print

```lua
z.log("text")
z.log.info("text")
z.log.warn("text")
z.log.error("text")
print("text")
```

`z.log` — таблица с `__call`, поэтому её можно вызывать как функцию.
`print` — алиас информационного лога. Аргументы приводятся к строке и
соединяются пробелом. Искусственного лимита длины сообщения нет; в памяти
хранятся все записи, во вкладке Lua показываются все текущие записи.

Логи доступны и при загрузке скрипта, и в `create_move`, `paint`, `ui`,
`shutdown`.

## z.render

См. [render](render.md). Работает в `paint` и вне его: вне `paint` команды
откладываются до начала следующего кадра.

## z.ui и z.settings

Пользовательские окна и доступ к настройкам чита:

- [пользовательские окна](ui.md);
- [настройки](settings.md).
