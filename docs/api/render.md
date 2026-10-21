# Рендер

Все функции доступны только внутри `paint`:

```lua
z.events.paint:add(function()
    z.render.rect_filled(20, 20, 120, 40, { r = 20, g = 20, b = 28, a = 220 }, 6)
end)
```

Вне `paint` менеджер выбрасывает
`render is only available inside the paint event`. Рисование идёт в средний
слой xdraw, после встроенных оверлеев и до меню.

## Цвет

Поддерживаются два формата:

```lua
-- таблица, рекомендуется
{ r = 255, g = 80, b = 40, a = 255 }

-- упакованное целое 0xAARRGGBB
0xFFFF5028
```

В таблице отсутствующие поля заменяются на `255`; значения обрезаются до
`0..255`. Альфа `0` — полностью прозрачный, `255` — непрозрачный.

## Функции

| Функция | Описание |
|---|---|
| `z.render.rect_filled(x, y, w, h, color[, radius])` | заполненный прямоугольник |
| `z.render.rect(x, y, w, h, color[, thickness[, radius]])` | рамка |
| `z.render.line(x0, y0, x1, y1, color[, thickness])` | линия |
| `z.render.circle(x, y, radius, color[, thickness])` | окружность |
| `z.render.circle_filled(x, y, radius, color)` | заполненный круг |
| `z.render.text(x, y, text, color)` | текст |
| `z.render.world_to_screen(x, y, z)` | `x, y` или `nil` |
| `z.render.world_to_screen(vector)` | `x, y` или `nil` |
| `z.render.measure_text(text)` | `width, height` |
| `z.render.screen_size()` | `width, height` |

Координаты — пиксели от левого верхнего угла. `w`/`h` и радиусы — числа с
плавающей точкой; отрицательные размеры не нормализуются.

## Примеры

### Плашка

```lua
z.events.paint:add(function()
    local color_bg = { r = 12, g = 12, b = 18, a = 225 }
    local color_line = { r = 139, g = 156, b = 255, a = 255 }

    z.render.rect_filled(20, 20, 220, 52, color_bg, 6)
    z.render.rect(20, 20, 220, 52, color_line, 1, 6)
    z.render.text(34, 32, "ZantiKira", color_line)
end)
```

### Индикатор

```lua
z.events.paint:add(function()
    local w, h = z.render.screen_size()
    local cx, cy = w * 0.5, h * 0.5

    z.render.circle(cx, cy, 18, { r = 255, g = 255, b = 255, a = 180 }, 1.5)
    z.render.circle_filled(cx, cy, 3, { r = 255, g = 80, b = 80, a = 255 })
end)
```

### Текст по центру

```lua
z.events.paint:add(function()
    local text = "hello"
    local tw, th = z.render.measure_text(text)
    local sw, sh = z.render.screen_size()

    z.render.text(
        (sw - tw) * 0.5,
        (sh - th) * 0.5,
        text,
        { r = 255, g = 255, b = 255, a = 255 }
    )
end)
```

## World to screen

```lua
z.events.paint:add(function()
    local x, y = z.render.world_to_screen(120.0, -40.0, 64.0)

    if x and y then
        z.render.circle_filled(x, y, 4.0, { r = 255, g = 80, b = 80, a = 255 })
    end
end)
```

Можно передать вектор из `z.math`:

```lua
local point = z.math.vec3(120.0, -40.0, 64.0)
local x, y = z.render.world_to_screen(point)
```

Функция возвращает `nil`, если точка за камерой или матрица ещё не обновлена.
Работает только внутри `paint`.

## Ограничения рендера

- Нет clip/scissor, градиентов, картинок и собственных шрифтов.
- Нет `z.render` в `create_move` или при загрузке скрипта.
- `paint` работает и при закрытом меню; проверяй `z.api.script`, если нужно
  рисовать только для конкретного скрипта.
- Текст рисуется встроенным шрифтом xdraw; размер шрифта не настраивается.
