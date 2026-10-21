# math

`z.math` — отдельная таблица, стандартная `math` не изменяется.

## Форматы

Вектор:

```lua
{ x = 1.0, y = 2.0, z = 3.0 }   -- плюс [1], [2], [3]
```

Угол:

```lua
{ pitch = 10.0, yaw = -45.0, roll = 0.0 } -- плюс [1], [2], [3]
```

Углы — в градусах, как в `cmd:get_view_angles()`.

Функции принимают таблицы; `vec3` и `angle` создают их. Поля можно не указывать —
недостающие значения считаются нулями.

## Функции

| Function | Return | Description |
|---|---|---|
| `z.math.vec3(x, y, z)` | vector | создать вектор |
| `z.math.angle(pitch, yaw[, roll])` | angle | создать угол |
| `z.math.angle_vectors(angle)` | forward, right, up | три направления |
| `z.math.vector_angles(vector)` | angle | направление → углы |
| `z.math.calc_angle(src, dst)` | angle | угол из `src` в `dst` |
| `z.math.calc_fov(a, b)` | number | разница углов в градусах |
| `z.math.normalize_angle(angle)` | angle | нормализовать pitch/yaw, roll = 0 |
| `z.math.normalize_yaw(yaw)` | number | привести yaw к `-180..180` |
| `z.math.distance(a, b)` | number | расстояние между точками |
| `z.math.length(vector)` | number | длина вектора |

`calc_angle` принимает **позиции**, а не углы. `calc_fov` принимает углы.

## Примеры

### Направление взгляда

```lua
local angles = z.math.angle(0.0, 90.0, 0.0)
local forward, right, up = z.math.angle_vectors(angles)

z.log(("forward: %.2f %.2f %.2f"):format(forward.x, forward.y, forward.z))
```

### Угол до точки

```lua
local eye = z.math.vec3(100.0, 200.0, 64.0)
local target = z.math.vec3(400.0, -50.0, 72.0)
local aim = z.math.calc_angle(eye, target)

z.log(("pitch %.1f yaw %.1f"):format(aim.pitch, aim.yaw))
```

### FOV

```lua
local current = z.math.angle(0.0, 45.0, 0.0)
local target = z.math.angle(2.0, 47.0, 0.0)
local fov = z.math.calc_fov(current, target)
```

### World to screen

```lua
z.events.paint:add(function()
    local position = z.math.vec3(0.0, 0.0, 0.0)
    local x, y = z.render.world_to_screen(position)

    if x then
        z.render.circle_filled(x, y, 4.0, { r = 255, g = 80, b = 80, a = 255 })
    end
end)
```

`world_to_screen` работает и вне `paint`; в этом случае используется последняя
матрица кадра. Возвращает `nil`, если точка за камерой или камера ещё не
обновлена.
