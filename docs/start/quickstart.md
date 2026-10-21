# Быстрый старт

## 1. Создай файл

```text
%LOCALAPPDATA%\zantikira\scripts\hello.lua
```

```lua
z.log("hello from " .. z.api.script)

z.events.paint:add(function()
    local w, h = z.render.screen_size()
    z.render.text(18, h - 34, "zantikira lua", {
        r = 255, g = 255, b = 255, a = 220
    })
end)
```

## 2. Добавь в конфиг

Открой **Lua**, введи `hello.lua`, нажми `add`, затем `reload`. В логе появится
`hello from <абсолютный путь>`.

Если скрипт не появился:

- проверь, что глобальный `enable scripting` включён;
- проверь расширение `.lua` и что файл не пустой;
- посмотри лог во вкладке Lua;
- убедись, что путь не выходит за папку скриптов;
- проверь, что путь включён (`enabled = true`).

## 3. Bunny hop

```lua
z.events.create_move:add(function(cmd)
    cmd:set_button(z.buttons.jump, true)
end)
```

`set_button` меняет только указанный бит. Остальные кнопки, которые нажал игрок,
сохраняются.

## 4. Движение

```lua
z.events.create_move:add(function(cmd)
    cmd:set_forward_move(450)
    cmd:set_left_move(0)
end)
```

Любой `set_*_move` помечает тик как script-controlled: встроенное финальное
subtick-обнуление movement пропускается. Это не отключает ragebot/legit/autostop
и не меняет 32-slice autostrafe; скрипт просто пишет поверх готового usercmd.

## 5. View angles

```lua
z.events.create_move:add(function(cmd)
    local a = cmd:get_view_angles()
    if a then
        cmd:set_view_angles_silent(a.pitch, a.yaw - 2.0, a.roll)
    end
end)
```

`set_view_angles` дополнительно двигает камеру. `set_view_angles_silent` меняет
только usercmd.

## 6. Снимки игроков

```lua
z.events.paint:add(function()
    for _, player in ipairs(z.entitylist.players()) do
        if not player.local and player.origin then
            local x, y = z.render.world_to_screen(player.origin)
            if x and y then
                z.render.text(x, y, ("%s | %d hp"):format(
                    player.name or "unknown", player.health
                ), { r = 255, g = 80, b = 80, a = 255 })
            end
        end
    end
end)
```

Кости, хитбоксы и видимость:

```lua
z.events.paint:add(function()
    for _, player in ipairs(z.entitylist.players()) do
        if not player.local then
            local head = z.entitylist.bone(player.index, "head")
            local can_see = z.entitylist.visible(player.index, "head")
            z.log(player.index, head and head.position, can_see)
        end
    end
end)
```

Подробности: [entitylist](../api/entitylist.md).

## 7. Рендер вне paint

`z.render` больше не привязан к `paint`. Вне его команды попадают в отложенную
очередь и появляются в начале следующего кадра:

```lua
z.events.create_move:add(function()
    z.render.text(20, 20, "deferred", { r = 255, g = 255, b = 255, a = 255 })
end)
```

Если нужен полный контроль над порядком, рисуй в `paint` или собирай
`z.render.new_list()`. Подробности:
[render](../api/render.md).

## 8. Prediction и autowall

```lua
z.events.create_move:add(function(cmd)
    local me = z.entitylist.local_player()
    if not me or not me.alive or not me.eye_position then
        return
    end

    for _, player in ipairs(z.entitylist.players()) do
        if not player.local and player.origin then
            local result = z.combat.autowall(me.eye_position, player.origin, player.index)
            if result.hit and result.damage > 0 then
                z.log(player.name, result.damage)
            end
        end
    end
end)
```

Autowall и trace дорогие: не вызывай их для каждой кости каждого игрока каждый
тик. Подробности: [combat](../api/combat.md) и [engine](../api/engine.md).

## 9. Сеть и ассеты

```lua
local response, err = z.net.http("https://example.com/", { timeout = 3000 })
if response and response.ok then
    z.log(#response.body)
end

local image = nil

z.events.paint:add(function()
    if not image then
        local image_err
        image, image_err = z.assets.load_image("images/logo.png")
        if not image then
            z.log.error(image_err)
        end
    end

    if image then
        z.render.image(20, 20, 64, 64, image)
    end
end)
```

Подробности: [network](../api/network.md) и [assets](../api/assets.md).

## 10. Перезагрузка из Lua

```lua
z.events.ui:add(function()
    z.ui.begin_window("script control", 40, 40, 260, 120)
    if z.ui.button("reload all", 120, 24) then
        z.scripts.reload()
    end
    z.ui.end_window()
end)
```

Перезагрузка выполняется вне активного callback. Подробности:
[scripts](../api/scripts.md).

## 11. FFI

```lua
local ffi = require("ffi") -- только при allow ffi
ffi.cdef[[
    int MessageBoxA(void* hwnd, const char* text, const char* caption, unsigned type);
]]
ffi.C.MessageBoxA(nil, "hello", "z", 0)
```

При выключенном FFI этот скрипт не загрузится: `require` выбросит ошибку, она
попадёт в лог, остальные скрипты продолжат работу.

## Частые ошибки

| Message | Cause |
|---|---|
| `module 'ffi' is unavailable` | FFI выключен |
| `render is unavailable before the game renderer is initialized` | рендер вызван до создания D3D11-устройства |
| `command is only valid while create_move is running` | сохранённый `cmd` использован после callback |
| `script must be inside ...` | путь выходит за папку скриптов |
| `only .lua scripts are allowed` | неверное расширение |
| `script file does not exist` | файл не найден |
| `script is empty` | пустой файл |
