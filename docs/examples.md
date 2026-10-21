# Примеры

## Bunny hop

```lua
z.events.create_move:add(function(cmd)
    cmd:set_button(z.buttons.jump, true)
end)
```

## Индикатор в углу

```lua
z.events.paint:add(function()
    local w, h = z.render.screen_size()
    local x, y = w - 230, h - 62

    z.render.rect_filled(x, y, 210, 42, {
        r = 10, g = 10, b = 16, a = 220
    }, 6)

    z.render.text(x + 12, y + 12, "zantikira", {
        r = 139, g = 156, b = 255, a = 255
    })
end)
```

## Индикатор локального игрока

```lua
z.events.paint:add(function()
    if not z.engine.is_in_game() then
        return
    end

    local player = z.entitylist.local_player()
    if not player or not player.alive then
        return
    end

    local text = ("%s | %d hp | %s"):format(
        player.name or "unknown",
        player.health,
        player.weapon or "none"
    )

    z.render.text(20, 20, text, { r = 255, g = 255, b = 255, a = 255 })
end)
```

`local_player()` возвращает снимок; данные не обновляются сами, поэтому вызывай
его внутри `paint` или `create_move` заново, когда нужен свежий кадр.

## ESP по игрокам

```lua
z.events.paint:add(function()
    for _, player in ipairs(z.entitylist.players()) do
        if not player.local and player.origin then
            local x, y = z.render.world_to_screen(player.origin)
            if x and y then
                local head = z.entitylist.bone(player.index, "head")
                if head then
                    local hx, hy = z.render.world_to_screen(head.position)
                    if hx and hy then
                        local color = { r = 255, g = 80, b = 80, a = 255 }
                        z.render.text(hx, hy - 16, player.name or "unknown", color)
                    end
                end

                z.render.text(x, y, ("%d hp"):format(player.health), {
                    r = 255, g = 255, b = 255, a = 255
                })
            end
        end
    end
end)
```

`z.entitylist.bone()` принимает controller index из `players()` и имя кости.
Кости и хитбоксы могут быть недоступны для dormant-игроков.

## Видимость и хитбоксы

```lua
local player = z.entitylist.players()[1]
if player then
    local visible = z.entitylist.visible(player.index, "head")
    local head = z.entitylist.hitbox(player.index, 0)
    z.log(visible, head and head.hitgroup_name)
end
```

`visible()` проверяет одну точку, а не всё тело. Для полной проверки перебирай
хитбоксы и делай `z.trace.hull` с габаритами.

## Override движения

```lua
local speed = 450.0

z.events.create_move:add(function(cmd)
    local a = cmd:get_view_angles()
    if not a then
        return
    end

    local yaw = math.rad(a.yaw)

    cmd:set_forward_move(math.cos(yaw) * speed)
    cmd:set_left_move(-math.sin(yaw) * speed)
end)
```

`set_*_move` отменяет финальное subtick-обнуление base movement на этом тике.
Не вызывай их без необходимости: встроенный autostrafe уже пишет движение.

## Autowall и backtrack

```lua
z.events.create_move:add(function()
    local me = z.entitylist.local_player()
    if not me or not me.alive or not me.eye_position then
        return
    end

    for _, player in ipairs(z.entitylist.players()) do
        if not player.local then
            local record = z.combat.backtrack(player.index)
            if record then
                for _, bone in ipairs(record.bones) do
                    if bone.name == "head" then
                        local result = z.combat.autowall(
                            me.eye_position, bone.position, player.index)

                        if result.hit and result.damage > 0 then
                            z.log(player.name, result.damage, result.hitgroup_name)
                        end
                        break
                    end
                end
            end
        end
    end
end)
```

`autowall` использует текущее оружие и lag compensation. Это дорого; ограничивай
число целей и точек. Подробности: [combat](api/combat.md).

## Prediction

```lua
z.events.create_move:add(function(cmd)
    local ok = z.engine.predict(function()
        local me = z.entitylist.local_player()
        if me and me.origin then
            z.log(("predicted z = %.2f"):format(me.origin.z))
        end
    end)

    if not ok then
        z.log.warn("prediction failed")
    end
end)
```

Callback выполняется, пока игра находится в предсказанном состоянии. Не вызывай
prediction внутри prediction. Вне `create_move` передай `cmd` явно:
`z.engine.predict(fn, cmd)`.

## Draw-list и градиент

```lua
local panel = z.render.new_list()

panel:rect_filled_gradient(0, 0, 260, 80,
    { r = 80, g = 110, b = 255, a = 230 },
    { r = 140, g = 80, b = 255, a = 230 },
    { r = 255, g = 80, b = 160, a = 230 },
    { r = 40, g = 30, b = 80, a = 230 },
    10)

panel:push_clip(10, 10, 240, 60)
panel:text(18, 22, "draw-list", { r = 255, g = 255, b = 255, a = 255 })
panel:pop_clip()

z.events.paint:add(function()
    z.render.submit(panel)
end)
```

Список можно собрать заранее и отправлять каждый кадр. Методы списка повторяют
`z.render`, но не содержат `measure_text`, `screen_size` и `world_to_screen`.

## Сеть

```lua
z.events.ui:add(function()
    z.ui.begin_window("net", 40, 40, 320, 120)

    if z.ui.button("request", 120, 24) then
        local response, err = z.net.http("https://example.com/", {
            method = "GET",
            timeout = 3000,
        })

        if response and response.ok then
            z.log("HTTP", response.status, #response.body)
        else
            z.log.error(err or "request failed")
        end
    end

    z.ui.end_window()
end)
```

HTTP синхронный: кнопка в UI заблокирует поток до ответа или таймаута. Для
периодических запросов лучше отдельный поток через FFI, но это уже
ответственность скрипта.

## Ассет и шрифт

```lua
local logo = nil

z.events.paint:add(function()
    if not logo then
        logo = z.assets.load_image("images/logo.png")
    end

    if logo then
        z.render.image(20, 20, 64, 64, logo)
    end
end)
```

## Настройки из скрипта

```lua
for _, entry in ipairs(z.settings.list("ragebot - rifle")) do
    z.log(("%s.%s = %s"):format(entry.category, entry.name, entry.type))
end

local hitchance = z.settings.get("ragebot - rifle", "hit chance")
z.settings.set("ragebot - rifle", "hit chance", math.min(hitchance + 5, 100))
```

Custom-поля тоже доступны:

```lua
local agents = z.settings.get("changer", "agents")
agents.ct = 0
agents.t = 0
z.settings.set("changer", "agents", agents)
```

Подробности: [settings](api/settings.md).

## Список скриптов

```lua
local list = z.settings.get("lua", "script list")
for _, entry in ipairs(list) do
    if entry.path == "overlay.lua" then
        entry.enabled = true
    end
end
z.settings.set("lua", "script list", list) -- перезагрузка будет запрошена сама
```

Запись списка не проверяет существование файла: ошибка появится в логе при
загрузке.

## FFI: MessageBox

```lua
local ffi = require("ffi")

ffi.cdef[[
    int MessageBoxA(void* hwnd, const char* text, const char* caption, unsigned type);
]]

ffi.C.MessageBoxA(nil, "hello from lua", "z", 0)
```

Работает только при `allow ffi (unsafe)`. При выключенном FFI скрипт не
загрузится, ошибка появится в логе.

## FFI: чтение памяти

```lua
local ffi = require("ffi")

local function read_u32(address)
    return ffi.cast("uint32_t*", address)[0]
end

z.log(("u32 at 0x%X = %d"):format(0x1000, read_u32(0x1000)))
```

Это пример синтаксиса, а не безопасный код: произвольный адрес может быть
невалидным. Проверяй адреса и не запускай чужие FFI-скрипты.

## Своё окно

```lua
local enabled = true
local speed = 1.0
local mode = 1

z.events.ui:add(function()
    local sw, sh = z.ui.screen_size()
    z.ui.begin_window("my panel", sw - 360, 80, 320, 240, true)

    z.ui.text("zantikira lua", { r = 139, g = 156, b = 255, a = 255 })
    z.ui.separator()

    enabled = z.ui.checkbox("enabled", enabled)
    speed = z.ui.slider("speed", speed, 0.1, 10.0)
    mode = z.ui.combo("mode", mode, { "off", "legacy", "strict" }, 160)

    z.ui.end_window()
end)
```

Позиция окна сохраняется между кадрами. Окно нужно закрыть до возврата из
обработчика; `z.ui` работает только в `ui`.

## Конфиг списка

```json
{
  "version": 2,
  "fields": {
    "7605c6c2": [
      { "path": "bhop.lua", "enabled": true },
      { "path": "overlay.lua", "enabled": true }
    ]
  }
}
```

Обычно этот список редактируется через вкладку **Lua**.

## Перезагрузка из UI

```lua
z.events.ui:add(function()
    z.ui.begin_window("scripts", 40, 40, 280, 120)

    if z.ui.button("reload all", 120, 24) then
        z.scripts.reload()
    end

    z.ui.end_window()
end)
```
