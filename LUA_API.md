# ZantiKira Lua API

LuaJIT 2.1, FFI включён в сборку. API написано с нуля; документация Fatality
использовалась только как ориентир по смыслу событий и методов.

> Это одностраничная версия. Сайт с навигацией и поиском лежит в
> [`docs/index.md`](docs/index.md); запуск: `mkdocs serve`.

## Запуск скриптов

Скрипты **не сканируются** и не запускаются автоматически из папки. Источник
истины — только список в активном конфиге. В `.cfg` это custom-поле
`lua.script list`; ключ в JSON — `7605c6c2`:

```json
{
  "version": 2,
  "fields": {
    "7605c6c2": [
      { "path": "my_script.lua", "enabled": true },
      { "path": "C:\\Users\\me\\AppData\\Local\\zantikira\\scripts\\other.lua", "enabled": false }
    ]
  }
}
```

Глобальные ключи в `fields`: `24ab6cad` — `lua.lua scripts`, `7f0f0e0e` —
`lua.lua ffi`. Обычно их меняют через вкладку Lua, а не вручную.

- Относительный путь разрешается от `%LOCALAPPDATA%\zantikira\scripts`.
- Абсолютный путь допустим только внутри этой папки; выход через `..`,
  junction/symlink и путь с другим корнем отклоняется.
- Только `.lua`, файл не пустой и не больше 2 MiB.
- В списке максимум 128 записей, путь до 512 символов.
- `enabled` в конфиге выключает конкретный скрипт.
- Глобальный `lua scripts` (по умолчанию `false`) выключает весь рантайм.
- `lua ffi` (по умолчанию `false`) — глобальный переключатель FFI для конфига.
- Список перезагружается при загрузке/сбросе конфига, смене FFI и кнопке
  `reload`. Скрипт, добавленный в список, начнёт работу после reload.

Один Lua state на скрипт. Ошибка загрузки или выполнения закрывает только этот
state, пишется в лог, остальные скрипты продолжают работу. `shutdown` вызывается
при reload, выгрузке и смене конфига; после runtime-ошибки state закрывается
сразу.

## Библиотеки

### FFI выключен

Доступны базовая библиотека (`_G`, `coroutine`), `table`, `string`, `math`,
`bit`, `jit`. `print` перенаправлен в лог ZantiKira.

Недоступны: `io`, `os`, `debug`, `package`, `dofile`, `loadfile`, `ffi`.
`require` заменён заглушкой, которая всегда выбрасывает ошибку. Поэтому
`require("ffi")` при выключенном FFI — ошибка загрузки скрипта, а не тихий
fallback.

### FFI включён

Открыты стандартные библиотеки LuaJIT, включая `ffi`, `io`, `os`, `debug` и
`package`. `package.path` начинается с папки скриптов, поэтому `require` может
подключать локальные `.lua`-модули.

Это не песочница: скрипт с FFI может читать/писать память процесса и вызывать
нативные функции. Запуск выполняется полностью на риск пользователя.

## События

```lua
z.events.create_move:add(function(cmd) end)
z.events.paint:add(function() end)
z.events.ui:add(function() end)
z.events.shutdown:add(function() end)
```

- `create_move` — после встроенного movement и до финального subtick-zeroing.
  Аргумент `cmd` — временный userdata.
- `paint` — после встроенных оверлеев, до меню. Только здесь доступен
  `z.render`.
- `ui` — внутри существующей пары `xui::begin`/`xui::end`, после окна меню.
  Только здесь доступен `z.ui`; окна скриптов рисуются поверх меню и остаются
  на экране при закрытом меню.
- `shutdown` — перед закрытием state при reload/выгрузке/смене конфига.

Обработчики вызываются в порядке добавления. Один и тот же обработчик можно
добавить несколько раз.

## cmd

`cmd` действителен только во время текущего `create_move`. После возврата из
обработчика все его методы начинают выбрасывать ошибку; сохранять userdata
нельзя.

```lua
cmd:get_button(z.buttons.jump)
cmd:set_button(z.buttons.jump, true)

cmd:get_buttons()
cmd:set_buttons(bits)          -- заменить весь bitmask

local a = cmd:get_view_angles()
-- a.pitch, a.yaw, a.roll; также a[1], a[2], a[3]
cmd:set_view_angles(pitch, yaw, roll)
cmd:set_view_angles_silent(pitch, yaw, roll)

cmd:get_forward_move()
cmd:get_left_move()
cmd:get_up_move()
cmd:set_forward_move(value)
cmd:set_left_move(value)
cmd:set_up_move(value)

cmd:get_command_number()
cmd:get_tick()
```

`set_view_angles` двигает камеру, `set_view_angles_silent` меняет только
usercmd. Вызов любого `set_*_move` помечает тик как script-controlled: финальное
обнуление base movement в этом тике пропускается. Установка кнопок или углов
такой флаг не ставит.

## z.render

Координаты — пиксели, начало в левом верхнем углу. Цвет — таблица
`{ r = 255, g = 80, b = 40, a = 255 }` или упакованное целое в порядке
`0xAARRGGBB` (например, `0xFFFF5028`). Предпочтительна таблица.

```lua
z.render.rect_filled(x, y, w, h, color[, radius])
z.render.rect(x, y, w, h, color[, thickness[, radius]])
z.render.line(x0, y0, x1, y1, color[, thickness])
z.render.circle(x, y, radius, color[, thickness])
z.render.circle_filled(x, y, radius, color)
z.render.text(x, y, text, color)
local x, y = z.render.world_to_screen(world_x, world_y, world_z)
local x, y = z.render.world_to_screen(vector)
local w, h = z.render.measure_text(text)
local sw, sh = z.render.screen_size()
```

Вне `paint` функции рендера выбрасывают ошибку. `world_to_screen` возвращает
`nil`, если точка за камерой или матрица ещё не обновлена.

## z.math

Вектор — `{ x, y, z }` (плюс `[1]`, `[2]`, `[3]`), угол —
`{ pitch, yaw, roll }` (плюс индексы). Стандартная `math` не меняется.

```lua
z.math.vec3(x, y, z)
z.math.angle(pitch, yaw[, roll])
local forward, right, up = z.math.angle_vectors(angle)
z.math.vector_angles(vector)
z.math.calc_angle(src, dst)
z.math.calc_fov(angle_a, angle_b)
z.math.normalize_angle(angle)
z.math.normalize_yaw(yaw)
z.math.distance(a, b)
z.math.length(vector)
```

`calc_angle` принимает позиции, `calc_fov` — углы. `normalize_angle` обнуляет
roll.

## z.globals

```lua
z.globals.curtime()      -- number или nil
z.globals.tick_count()   -- integer или nil
z.globals.is_in_game()   -- boolean
z.globals.frame_time()   -- delta кадра
z.globals.fps()          -- FPS xdraw
z.globals.snapshot()     -- table
```

`curtime` и `tick_count` возвращают `nil`, пока `global_vars` недоступны.

## z.engine

```lua
z.engine.is_connected()      -- интерфейсы движка инициализированы
z.engine.is_in_game()        -- есть локальный pawn/observer
z.engine.client_cmd(text)    -- выполнить клиентскую команду
```

`is_connected` не гарантирует активное сетевое соединение. `client_cmd`
принимает 1–512 байт без NUL, выполняется сразу и требует инициализированный
engine client. Это произвольная клиентская команда, включая `quit`/`disconnect`;
для движения используй `cmd`, а не `client_cmd`.

## z.entitylist

`z.entitylist` отдаёт снимки, а не сырые указатели. Каждый вызов читает данные
заново; таблицу можно сохранить, но она не обновляется сама.

```lua
local players = z.entitylist.players()
-- массив живых игроков, отсортирован по index
-- локальный игрок тоже входит в массив, у него local = true
```

`players()` возвращает пустую таблицу, если живых игроков нет. Мёртвые игроки
не попадают в список. `index` — индекс controller-сущности, не pawn и не слот
scoreboard.

```lua
local samples = z.entitylist.history(index, 16)
-- motion-снимки одного игрока, от старых к новым
-- index — controller index из players(), limit 1..64
```

История собирается автоматически раз в `create_move`, пока включён scripting.
В ней только живые игроки и только motion-поля: `index`, `tick`, `time`,
`health`, `team`, `origin`, `eye_position`, `velocity`, `abs_velocity`,
`flags`, `on_ground`, `ducking`, `sim_time`. При смене локального/наблюдаемого
pawn история сбрасывается; снимки старого pawn с тем же index не возвращаются
после респавна.

```lua
local p = z.entitylist.local_player()
-- nil, пока нет controller/pawn

p.alive            -- boolean
p.observer         -- boolean, активен observer-режим
p.scoped           -- boolean
p.health           -- integer
p.team             -- integer
p.armor            -- integer
p.ping             -- integer
p.name             -- string или nil
p.weapon           -- string без "weapon_" или nil
p.weapon_id        -- item definition index, 0 если неизвестно
p.ammo             -- integer
p.max_ammo         -- integer
p.origin           -- vector или nil
p.eye_position     -- vector или nil
p.velocity         -- vector или nil, m_vecVelocity
p.abs_velocity     -- vector или nil, m_vecAbsVelocity
p.flags            -- integer, m_fFlags
p.on_ground        -- boolean
p.ducking          -- boolean
p.sim_time         -- number или nil, m_flSimulationTime
```

У элементов `players()` дополнительно есть `index` и `local`; `observer` там
всегда `false`. `sim_time`, `velocity`, `abs_velocity`, `origin` и
`eye_position` могут отсутствовать, пока данные не готовы. `players()` даёт
данные для собственной оценки, но не выполняет engine prediction, trace или
backtrack.

Алиасы: `z.entitylist.get_local_player()` и
`z.entitylist.get_local_player_pawn()` возвращают ту же таблицу.

```lua
z.entitylist.get_local_player_name()  -- string или nil
z.entitylist.get_local_weapon()       -- { name, id, ammo, max_ammo } или nil
z.entitylist.get_local_weapon_name()  -- string или nil
z.entitylist.get_local_weapon_index() -- integer или nil
```

В observer-режиме поля описывают наблюдаемого игрока. Motion-поля, `origin` и
`eye_position` могут отсутствовать, пока данные ещё не готовы.

## z.trace

Один луч или hull через игровой trace-менеджер. Без FFI и без raw-указателей.

```lua
local tr = z.trace.line(start, finish, {
    skip_local = true,   -- по умолчанию true
    mask = 0x1c3003,     -- по умолчанию
    layer = 4,           -- по умолчанию
})

tr.hit                 -- boolean
tr.fraction            -- number, 0..1
tr.position            -- vector
tr.normal              -- vector
tr.contents            -- integer
tr.all_solid           -- boolean
tr.hit_player          -- boolean
tr.hit_player_index    -- controller index или nil
```

`z.trace.hull(start, finish, mins, maxs, options?)` делает trace с габаритами.
`hit_player_index` совпадает с `index` из `z.entitylist.players()`.
`skip_local` в observer-режиме пропускает наблюдаемый pawn.

Это не autowall и не backtrack: один вызов — один луч или hull, без penetration.
Результат — копия данных, ссылок на сущности нет.

## z.ui

Immediate-mode окна в стиле ImGui на существующем `xui`. Работают только в
`z.events.ui`; это не второй Dear ImGui context и не `z.render`.

```lua
z.events.ui:add(function()
    local sw, sh = z.ui.screen_size()
    z.ui.begin_window("my panel", sw - 360, 80, 320, 240, true)

    z.ui.text("hello")
    z.ui.separator()

    local enabled = z.ui.checkbox("enabled", true)
    local speed = z.ui.slider("speed", 1.0, 0.1, 10.0)
    local mode = z.ui.combo("mode", 1, { "off", "legacy", "strict" }, 160)
    local name = z.ui.text_input("name", "player", 64, "player name")

    if z.ui.button("apply", 100, 24) then
        z.log(("%s %s %d %.2f"):format(name, tostring(enabled), mode, speed))
    end

    z.ui.end_window()
end)
```

```lua
z.ui.begin_window(title, x, y, w, h[, resizable])
z.ui.end_window()
z.ui.set_window_rect(title, x, y, w, h)

z.ui.text(text[, color])
z.ui.separator()
z.ui.same_line([offset])
z.ui.spacing([amount])
z.ui.button(label[, width, height])
z.ui.checkbox(label, value)
z.ui.slider(label, value, min, max[, format])
z.ui.slider_int(label, value, min, max[, format])
z.ui.combo(label, index, items[, width])
z.ui.text_input(label, value[, max_len[, hint]])
z.ui.screen_size()
z.ui.measure_text(text)
```

Позиция и размер окна сохраняются в состоянии скрипта. `combo` принимает
индекс с `1` и хранит выбор; `text_input` возвращает текущий буфер. Суффикс
`##id` скрывает часть подписи и делает ID уникальным. Окно нужно закрыть до
возврата из callback; незакрытое окно закроется принудительно с
предупреждением в логе. Состояние сбрасывается при reload. При закрытом меню
окно остаётся на экране, но клики/ввод обычно недоступны, пока игра держит
курсор.

## z.settings

Доступ к primitive-полям того же реестра, что и меню/конфиг. Raw-указателей
нет; custom-поля (скины, агенты, список скриптов) не читаются и не пишутся.

```lua
for _, entry in ipairs(z.settings.list("ragebot - rifle")) do
    z.log(("%s.%s = %s"):format(entry.category, entry.name, entry.type))
end

z.settings.exists("ragebot - rifle", "hit chance") -- boolean
z.settings.type("ragebot - rifle", "hit chance")   -- "int" или nil
local value = z.settings.get("ragebot - rifle", "hit chance")
z.settings.set("ragebot - rifle", "hit chance", 65)
```

Поддерживается короткая форма `z.settings.get("category.name")` и
`z.settings.set("category.name", value)`. Типы: `bool`, `int`, `float`,
`color`, `vec3`, `bool[]`, `string`, `custom`. Для `xui::setting` запись
обновляет и toggle-bind; hold-bind может перезаписать значение на следующем
кадре. Неизвестное имя, неверный тип и запись custom-поля — ошибка Lua.

## Время, лог, константы

```lua
z.time.now()    -- Unix time, секунды (float)
z.time.tick()   -- секунды с запуска Windows
z.time.frame()  -- delta последнего кадра, секунды

z.log("text")
z.log.info("text")
z.log.warn("text")
z.log.error("text")
print("text")       -- то же, что z.log
```

`z.log` и `print` принимают несколько аргументов, приводя их к строке через
пробел. Сообщения попадают во вкладку Lua и в диагностический лог.

`z.api`:

```lua
z.api.version  -- "0.6.0"
z.api.game     -- "cs2"
z.api.script   -- абсолютный путь загруженного скрипта
z.api.ffi      -- true, если FFI включён
```

`z.buttons`: `attack`, `jump`, `duck`, `forward`, `back`, `use`, `left`,
`right`, `move_left`, `move_right`, `second_attack`, `reload`, `sprint`,
`scoreboard`, `zoom`, `look_at_weapon`.

## Примеры

### Bunny hop

```lua
z.events.create_move:add(function(cmd)
    cmd:set_button(z.buttons.jump, true)
end)
```

### Overlay

```lua
z.events.paint:add(function()
    z.render.rect_filled(20, 20, 180, 42, { r = 18, g = 18, b = 22, a = 220 }, 6)
    z.render.text(32, 32, "zantikira lua", { r = 255, g = 255, b = 255, a = 255 })
end)
```

### Локальный игрок

```lua
z.events.paint:add(function()
    local p = z.entitylist.local_player()
    if not p or not p.alive then
        return
    end

    z.render.text(20, 20, ("%s | %d hp | %s"):format(
        p.name or "unknown", p.health, p.weapon or "none"
    ), { r = 255, g = 255, b = 255, a = 255 })
end)
```

### FFI

```lua
local ffi = require("ffi") -- работает только при включённом FFI
ffi.cdef[[ int MessageBoxA(void*, const char*, const char*, unsigned); ]]
ffi.C.MessageBoxA(nil, "hello", "z", 0)
```

## Ограничения текущего P0

Полного списка не-игровых сущностей, autowall/backtrack, HTTP и custom-полей
настроек пока нет. `z.entitylist` отдаёт снимки локального игрока, живых
игроков и короткую motion-историю; `z.trace` даёт один луч/hull, но не engine
prediction и не penetration. `z.ui` — базовый набор
виджетов без деревьев, таблиц, color picker и keybind. `z.settings` работает
только с primitive-полями. Скрипты запускаются только из списка конфига;
облака, allowlist и подписи нет. Любая ошибка скрипта изолирована его state, но
FFI-скрипт при включённом флаге имеет полный доступ к процессу,
`z.engine.client_cmd` может выполнить произвольную клиентскую команду, а
`z.settings.set` меняет настройки чита без отдельного allowlist.
