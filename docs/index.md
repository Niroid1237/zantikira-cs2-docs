# ZantiKira Lua API

LuaJIT 2.1 · FFI по умолчанию выключен · один state на скрипт

API для CS2-чита ZantiKira, версия **0.7.0**. Скрипты запускаются только из
явного списка в активном конфиге, без сканирования папки и без облака.

!!! warning "Скрипт — это код в процессе игры"
    FFI даёт прямой доступ к памяти и native-функциям. Даже без FFI доступны
    сеть, чтение файлов как ассетов, произвольные клиентские команды и запись
    настроек чита. Запускай только свой код или код, которому полностью
    доверяешь.

## Что доступно

| Section | Features |
|---|---|
| [Events](api/events.md) | `create_move`, `paint`, `ui`, `shutdown` |
| [Command](api/command.md) | кнопки, view angles, forward/left/up move, номер команды, tick |
| [Entity list](api/entitylist.md) | живые игроки, все не-игровые сущности, кости, хитбоксы, видимость, motion-история |
| [Combat](api/combat.md) | autowall, lag compensation records, backtrack-снимки |
| [Trace](api/trace.md) | луч и hull с fraction, позицией, нормалью и индексом игрока |
| [Engine](api/engine.md) | состояние движка, client commands, prediction |
| [Math](api/math.md) | векторы, углы, FOV, distance |
| [Render](api/render.md) | примитивы, градиенты, blur, clip, изображения, шрифты, draw-list, deferred render |
| [Assets](api/assets.md) | PNG/GIF/SVG, пользовательские шрифты, image/font-токены |
| [Network](api/network.md) | синхронный HTTP, DNS, TCP, UDP |
| [UI](api/ui.md) | ImGui-подобные окна, checkbox, slider, combo, text input, кнопки |
| [Settings](api/settings.md) | primitive- и custom-поля: скины, агенты, список скриптов |
| [Scripts](api/scripts.md) | перезагрузка, папка скриптов |
| [Globals](api/globals.md) | `z.api`, `z.globals`, `z.time`, `z.buttons`, `z.log`, `print` |
| FFI | `require("ffi")` только при включённом чекбоксе |

## Быстрый старт

```lua
z.events.create_move:add(function(cmd)
    cmd:set_button(z.buttons.jump, true)
end)

z.events.paint:add(function()
    z.render.rect_filled(20, 20, 190, 42, { r = 15, g = 15, b = 22, a = 220 }, 6)
    z.render.text(32, 32, "zantikira lua", { r = 255, g = 255, b = 255, a = 255 })
end)
```

1. Положи `.lua` в `%LOCALAPPDATA%\zantikira\scripts`.
2. Открой **Lua** в меню, включи `enable scripting`.
3. Добавь путь в список и нажми `reload`.
4. Для `require("ffi")` включи `allow ffi (unsafe)` — рантайм перезагрузится.

Своё окно рисуется из `ui`, а не из `paint`:

```lua
z.events.ui:add(function()
    z.ui.begin_window("hello", 40, 40, 280, 160, true)
    z.ui.text("zantikira lua")
    z.ui.end_window()
end)
```

Подробно: [Installation](start/installation.md) и [Quickstart](start/quickstart.md).

## Порядок вызова

```text
create_move:
  встроенный movement
  → Lua create_move
  → финальное subtick-обнуление (если Lua не заявил движение)
  → input apply

frame:
  → deferred Lua draw (то, что вызвано вне paint)
  → reload, если он был запрошен
  → Lua paint
  → меню и виджеты xdraw
```

`z.render` работает и вне `paint`: команды уходят в отложенный список и
появляются в начале следующего `on_present`. `cmd` живёт только до возврата из
`create_move`.

## Документация

- [Установка](start/installation.md) — папка скриптов, список, FFI.
- [API](api/events.md) — модули `z.*`.
- [Примеры](examples.md) — готовые сценарии.
- [Безопасность](security.md) — что реально доступно скрипту.
- [Ограничения](limits.md) — практические границы, а не искусственные лимиты.

Публичная копия: <https://niroid1237.github.io/zantikira-cs2-docs/>.

## Локальный просмотр сайта

```bash
pip install -r requirements-docs.txt
mkdocs serve
```

Сайт откроется на `http://127.0.0.1:8000`. Продакшен-сборка:

```bash
python -m mkdocs build --strict
```
