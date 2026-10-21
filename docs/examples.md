# Примеры

## Bunny hop

```lua
z.events.create_move:add(function(cmd)
    cmd:set_button(z.buttons.jump, true)
end)
```

## Индикатор в углу

```lua
local enabled = true

z.events.paint:add(function()
    if not enabled then
        return
    end

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

## Настройки из скрипта

```lua
for _, entry in ipairs(z.settings.list("ragebot - rifle")) do
    z.log(("%s.%s = %s"):format(entry.category, entry.name, entry.type))
end

local hitchance = z.settings.get("ragebot - rifle", "hit chance")
z.settings.set("ragebot - rifle", "hit chance", math.min(hitchance + 5, 100))
```

`z.settings` меняет тот же primitive-реестр, что и меню. Custom-поля
(списки скинов/агентов) через него недоступны.

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

Обычно этот список редактируется через вкладку **Lua**. После ручного изменения
файла загрузи конфиг заново или нажми `reload`.
