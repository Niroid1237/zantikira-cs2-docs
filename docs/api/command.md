# cmd

`cmd` — userdata, который передаётся в `create_move`:

```lua
z.events.create_move:add(function(cmd)
    -- ...
end)
```

Он действителен только до возврата из обработчика. После callback все методы
выбрасывают `command is only valid while create_move is running`. Не сохраняй
`cmd` в глобальную таблицу, замыкание или поле объекта.

## Кнопки

| Method | Return | Description |
|---|---|---|
| `cmd:get_button(mask)` | boolean | установлен ли бит |
| `cmd:set_button(mask, pressed)` | nil | поставить/снять один бит |
| `cmd:get_buttons()` | integer | весь bitmask |
| `cmd:set_buttons(bits)` | nil | заменить весь bitmask |

```lua
cmd:set_button(z.buttons.jump, true)
cmd:set_button(z.buttons.duck, false)

local bits = cmd:get_buttons()
cmd:set_buttons(bits)
```

`set_buttons` заменяет состояние целиком, а не дополняет его. Для обычного
включения одной кнопки используй `set_button`: он не затрагивает остальные
клавиши игрока.

Высокие биты (`scoreboard`, `zoom`, `look_at_weapon`) лучше ставить через
`set_button`: стандартная библиотека `bit` в LuaJIT 32-битная и не удержит
1 << 33 и выше.

## View angles

| Method | Return | Description |
|---|---|---|
| `cmd:get_view_angles()` | table или nil | `pitch`, `yaw`, `roll` |
| `cmd:set_view_angles(pitch, yaw[, roll])` | nil | записать углы и подвинуть камеру |
| `cmd:set_view_angles_silent(pitch, yaw[, roll])` | nil | записать только usercmd |

```lua
local a = cmd:get_view_angles()
if a then
    cmd:set_view_angles_silent(a.pitch, a.yaw - 2.0, a.roll)
end
```

Таблица содержит и именованные поля, и индексы:

```lua
local a = cmd:get_view_angles()
local pitch, yaw, roll = a.pitch, a.yaw, a.roll
-- эквивалентно: a[1], a[2], a[3]
```

`get_view_angles` возвращает `nil`, если у команды ещё нет base-сообщения.
`set_view_angles`/`set_view_angles_silent` в этом случае выбрасывают ошибку.

## Движение

| Method | Return | Description |
|---|---|---|
| `cmd:get_forward_move()` | number | forwardmove |
| `cmd:get_left_move()` | number | leftmove |
| `cmd:get_up_move()` | number | upmove |
| `cmd:set_forward_move(value)` | nil | записать forwardmove |
| `cmd:set_left_move(value)` | nil | записать leftmove |
| `cmd:set_up_move(value)` | nil | записать upmove |

```lua
cmd:set_forward_move(450.0)
cmd:set_left_move(0.0)
```

Любой `set_*_move` помечает текущий тик как script-controlled. Финальное
subtick-обнуление base movement пропускается; встроенный autostrafe и pixelsurf
при этом не меняются — скрипт просто пишет поверх уже готового usercmd.

## Команда и tick

| Method | Return | Description |
|---|---|---|
| `cmd:get_command_number()` | integer | номер команды |
| `cmd:get_tick()` | integer | client tick из base-сообщения |

`get_tick` возвращает `0`, если base-сообщения нет.

## Полный пример

```lua
z.events.create_move:add(function(cmd)
    -- bhop
    cmd:set_button(z.buttons.jump, true)

    -- мягкий yaw только в usercmd, камера не дёргается
    local a = cmd:get_view_angles()
    if a then
        cmd:set_view_angles_silent(a.pitch, a.yaw - 1.5, a.roll)
    end

    -- движение, если нужно переопределить встроенное
    if cmd:get_tick() % 2 == 0 then
        cmd:set_forward_move(450.0)
    end
end)
```
