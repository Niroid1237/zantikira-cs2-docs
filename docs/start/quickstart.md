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
- убедись, что путь не выходит за папку скриптов.

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
    cmd:set_view_angles_silent(a.pitch, a.yaw - 2.0, a.roll)
end)
```

`set_view_angles` дополнительно двигает камеру. `set_view_angles_silent` меняет
только usercmd.

## 6. FFI

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

| Сообщение | Причина |
|---|---|
| `module 'ffi' is unavailable` | FFI выключен |
| `render is only available inside the paint event` | рендер вызван из `create_move` или при загрузке |
| `command is only valid while create_move is running` | сохранённый `cmd` использован после callback |
| `script must be inside ...` | путь выходит за папку скриптов |
| `only .lua scripts are allowed` | неверное расширение |
| `script is larger than 2 MiB` | файл слишком большой |
