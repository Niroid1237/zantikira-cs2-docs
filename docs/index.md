# ZantiKira Lua API

LuaJIT 2.1 · FFI · один state на скрипт

API для CS2-чита ZantiKira. Скрипты запускаются только из явного списка в
активном конфиге, без сканирования папки и без облака.

!!! warning "FFI — это полный доступ к процессу"
    При включённом FFI скрипт может читать и писать память, вызывать
    `LoadLibrary`, `MessageBoxA` и любые другие native-функции. Запускай только
    свой код или код, которому полностью доверяешь.

## Что уже доступно

| Section | Features |
|---|---|
| [Events](api/events.md) | `create_move`, `paint`, `ui`, `shutdown` |
| [Command](api/command.md) | кнопки, view angles, forward/left/up move, номер команды, tick |
| [Entity list](api/entitylist.md) | снимки локального игрока и живых игроков, motion-история по тикам |
| [Trace](api/trace.md) | луч и hull с fraction, позицией, нормалью и индексом игрока |
| [Engine](api/engine.md) | `is_connected`, `is_in_game`, `client_cmd` |
| [Math](api/math.md) | векторы, углы, FOV, distance |
| [Render](api/render.md) | прямоугольники, линии, круги, текст, `world_to_screen`, размеры экрана |
| [UI](api/ui.md) | ImGui-подобные окна, checkbox, slider, combo, text input, кнопки |
| [Settings](api/settings.md) | чтение и запись primitive-настроек по `category, name` |
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
  встроенные оверлеи
  → Lua paint
  → widgets / меню
```

`paint` — единственное место, где работает `z.render`. `cmd` живёт только
до возврата из `create_move`.

## Ограничения P0

Сейчас есть снимки игроков, короткая история по тикам и trace-луч/hull, но нет
полного списка не-игровых сущностей, костей/хитбоксов, engine prediction,
autowall/backtrack и HTTP. Это сознательно: сначала стабильный
movement/render/log слой, потом остальное.
Полный список — в разделе [ограничения](limits.md), порядок модулей — в
[плане](roadmap.md).

## Локальный просмотр сайта

```bash
pip install -r requirements-docs.txt
mkdocs serve
```

Сайт откроется на `http://127.0.0.1:8000`. Для GitHub Pages добавлен workflow
`.github/workflows/docs.yml`: пуш в `main` или `master` собирает и публикует
`site/`.

В GitHub: **Settings → Pages → Build and deployment → Source: GitHub Actions**,
затем пуш в `main`/`master`. Если репозиторий называется иначе, при желании
раскомментируй `site_url` в `mkdocs.yml` и подставь свой адрес.
