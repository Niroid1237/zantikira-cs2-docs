# render

`z.render` рисует в средний слой xdraw поверх сцены и встроенных оверлеев, но
под меню. Внутри `paint` команды уходят в текущий кадр. Вне `paint` они
складываются в отложенный список и отправляются в начале следующего
`on_present` — то есть появляются на кадр позже.

```lua
z.events.paint:add(function()
    z.render.rect_filled(20, 20, 120, 40, { r = 20, g = 20, b = 28, a = 220 }, 6)
end)
```

Отложенный рендер работает при загрузке скрипта, в `create_move`, `ui` и
`shutdown`, но требует инициализированного D3D11-рендерера. Если устройство ещё
не создано, функция выбрасывает
`render is unavailable before the game renderer is initialized`.

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

## Примитивы

| Function | Description |
|---|---|
| `z.render.rect_filled(x, y, w, h, color[, radius])` | заполненный прямоугольник |
| `z.render.rect_filled_gradient(x, y, w, h, tl, tr, br, bl[, radius])` | градиент по четырём углам |
| `z.render.rect_filled_blurred(x, y, w, h[, tint[, radius]])` | размытие фона под прямоугольником |
| `z.render.rect(x, y, w, h, color[, thickness[, radius]])` | рамка |
| `z.render.line(x0, y0, x1, y1, color[, thickness])` | линия |
| `z.render.circle(x, y, radius, color[, thickness])` | окружность |
| `z.render.circle_filled(x, y, radius, color)` | заполненный круг |
| `z.render.triangle_filled(x0, y0, x1, y1, x2, y2, color)` | заполненный треугольник |
| `z.render.polyline(points, color[, closed[, thickness]])` | ломаная |
| `z.render.polyline_gradient(points, colors[, closed[, thickness]])` | ломаная с цветом на точку |
| `z.render.convex_filled(points, color)` | заполненный выпуклый многоугольник |
| `z.render.text(x, y, text, color[, font][, style])` | текст |
| `z.render.measure_text(text[, font])` | `width, height` |
| `z.render.screen_size()` | `width, height` |
| `z.render.world_to_screen(x, y, z)` | `x, y` или `nil` |
| `z.render.world_to_screen(vector)` | `x, y` или `nil` |

Координаты — пиксели от левого верхнего угла. `w`/`h` и радиусы — числа с
плавающей точкой. Заполненные примитивы с `w <= 0` или `h <= 0` пропускаются.
Отрицательные радиусы не нормализуются — используй `0`.

### Градиент

`rect_filled_gradient` принимает цвета в порядке **top-left, top-right,
bottom-right, bottom-left**. Радиус скругляет все четыре угла.

```lua
z.render.rect_filled_gradient(20, 20, 240, 64,
    { r = 80, g = 110, b = 255, a = 220 },
    { r = 140, g = 80, b = 255, a = 220 },
    { r = 255, g = 80, b = 160, a = 220 },
    { r = 40, g = 30, b = 80, a = 220 },
    8)
```

`polyline_gradient` берёт цвет для каждой точки. Если цветов меньше точек,
последний цвет повторяется.

### Размытие

```lua
z.render.rect_filled_blurred(20, 20, 320, 180, { r = 255, g = 255, b = 255, a = 180 }, 10)
```

`tint` по умолчанию белый непрозрачный. Размытие использует готовую цепочку
xdraw и координаты экрана. Это не дешёвый эффект: не вызывай его на десятки
прямоугольников каждый кадр.

### Точки

`points` — массив `{ x, y }` или плоский массив `{ x1, y1, x2, y2, ... }`.
Для polyline нужно минимум две точки, для `convex_filled` — минимум три.

```lua
local points = {
    { x = 40, y = 40 },
    { x = 160, y = 70 },
    { x = 90, y = 150 },
}

z.render.polyline(points, { r = 139, g = 156, b = 255, a = 255 }, true, 2)
z.render.convex_filled(points, { r = 20, g = 20, b = 30, a = 180 })
```

### Текст

`style` — `"normal"`, `"outlined"` или `"shadowed"`. Шрифт необязателен; без
него используется встроенный шрифт xdraw. Чтобы задать стиль без своего
шрифта, передай строку пятым аргументом.

```lua
z.render.text(20, 20, "normal", { r = 255, g = 255, b = 255, a = 255 })
z.render.text(20, 40, "outlined", { r = 255, g = 255, b = 255, a = 255 }, "outlined")
z.render.text(20, 60, "font", { r = 255, g = 255, b = 255, a = 255 }, font, "shadowed")
```

## Clip

```lua
z.render.push_clip(x, y, w, h)
z.render.push_clip_absolute(x, y, w, h)
z.render.pop_clip()
```

- `push_clip` пересекает новый прямоугольник с текущим clip.
- `push_clip_absolute` заменяет текущий clip, не пересекая с ним.
- `pop_clip` снимает последний clip. Лишний `pop_clip` ничего не делает.

Clip — это состояние draw-list. Внутри `paint` он действует до конца текущего
кадра или до `pop_clip`; в отложенном списке — до отправки накопленных команд.
Всегда закрывай clip, иначе обрежется чужой последующий рендер.

```lua
z.render.push_clip(20, 20, 160, 80)
z.render.text(20, 40, "обрезано", color)
z.render.pop_clip()
```

## Шрифты

```lua
z.render.push_font(font)
z.render.text(20, 100, "pushed font", color)
z.render.pop_font()
```

Стек шрифтов глобальный для xdraw. Он не привязан к скрипту: если не вызвать
`pop_font`, встроенные оверлеи и меню тоже начнут рисоваться этим шрифтом.
Не используй `push_font` вне `paint`, если не уверен, что `pop_font` будет
вызван в том же кадре.

Шрифты загружаются через [z.assets](assets.md).

## Изображения

```lua
z.render.image(x, y, w, h, image[, tint[, radius]])
z.render.image_uv(x, y, w, h, image, u0, v0, u1, v1[, tint])
```

`tint` — цвет или упакованное целое. `image_uv` принимает нормализованные UV в
диапазоне `0..1`.

```lua
z.render.image(20, 20, 64, 64, logo)
z.render.image_uv(92, 20, 64, 64, atlas, 0.0, 0.0, 0.5, 0.5)
```

Изображения и шрифты загружаются через [z.assets](assets.md).

## World to screen

```lua
local x, y = z.render.world_to_screen(120.0, -40.0, 64.0)
local x, y = z.render.world_to_screen(z.math.vec3(120.0, -40.0, 64.0))
```

Возвращает `nil`, если точка за камерой или матрица ещё не обновлена. Вне
`paint` используется последняя известная матрица кадра, поэтому координаты
могут отставать на один кадр.

## Draw-list

Draw-list позволяет собрать команды один раз и отправить их позже или
несколько раз.

```lua
local list = z.render.new_list()
list:rect_filled(20, 20, 200, 60, { r = 12, g = 12, b = 18, a = 220 }, 8)
list:rect(20, 20, 200, 60, { r = 139, g = 156, b = 255, a = 255 }, 1, 8)
list:text(32, 38, "cached draw-list", { r = 255, g = 255, b = 255, a = 255 })

z.events.paint:add(function()
    z.render.submit(list)   -- или list:submit()
end)
```

`z.render.create_list()` — алиас `new_list`. `z.render.submit(list)` — алиас
`list:submit()`.

### Методы списка

| Method | Description |
|---|---|
| `list:rect_filled(...)` | как `z.render.rect_filled` |
| `list:rect_filled_gradient(...)` | как `z.render.rect_filled_gradient` |
| `list:rect_filled_blurred(...)` | как `z.render.rect_filled_blurred` |
| `list:rect(...)` | как `z.render.rect` |
| `list:line(...)` | как `z.render.line` |
| `list:circle(...)` | как `z.render.circle` |
| `list:circle_filled(...)` | как `z.render.circle_filled` |
| `list:triangle_filled(...)` | как `z.render.triangle_filled` |
| `list:polyline(...)` | как `z.render.polyline` |
| `list:polyline_gradient(...)` | как `z.render.polyline_gradient` |
| `list:convex_filled(...)` | как `z.render.convex_filled` |
| `list:text(...)` | как `z.render.text`, включая шрифт и стиль |
| `list:image(...)` | как `z.render.image` |
| `list:image_uv(...)` | как `z.render.image_uv` |
| `list:push_clip(...)` | как `z.render.push_clip` |
| `list:push_clip_absolute(...)` | как `z.render.push_clip_absolute` |
| `list:pop_clip(...)` | как `z.render.pop_clip` |
| `list:clear()` | очистить вершины и команды |
| `list:append(other)` | добавить копию другого списка с пересчётом индексов |
| `list:submit()` / `list:draw()` | отправить в текущий кадр или очередь |
| `tostring(list)` | `draw_list(N vertices, M commands)` |

У методов списка нет `measure_text`, `screen_size`, `world_to_screen`,
`push_font` и `pop_font`. Для измерения используй глобальные функции.

```lua
local left = z.render.new_list()
left:rect_filled(0, 0, 80, 24, color)

local right = z.render.new_list()
right:append(left)
right:rect_filled(84, 0, 80, 24, color)

z.events.paint:add(function()
    z.render.submit(right)
end)
```

`submit` внутри `paint` добавляет список в текущий кадр сразу. Вне `paint`
команды попадают в отложенную очередь. Список можно отправить несколько раз;
он не очищается автоматически. `append` не забирает команды источника.

## Отложенный рендер

```lua
z.events.create_move:add(function()
    z.render.text(20, 20, "next frame", { r = 255, g = 255, b = 255, a = 255 })
end)
```

- Команды вне `paint` отправляются в начале следующего `on_present` и рисуются
  до Lua `paint` этого кадра.
- Если вызывать их каждый тик, в одном кадре может накопиться несколько
  вызовов; они все будут добавлены.
- Пока игра не представляет кадр (загрузка/свёрнутое окно), очередь растёт.
  Не рисуй тяжёлые сцены из `create_move` без ограничения.
- Clip входит в данные draw-list и применяется при отправке отложенных команд;
  незакрытый `push_clip` может обрезать последующие команды.
- `push_font` не является командой draw-list: он сразу меняет глобальный стек
  xdraw, даже вне `paint`. Всегда вызывай `pop_font`.
- Отложенный draw-list хранит ссылки на текстуры. Не освобождай изображение,
  пока команды не отправлены.

## Ограничения

- Нет произвольных вершинных шейдеров, 3D-примитивов и чтения backbuffer.
- Нет `z.render` до создания D3D11-устройства.
- Нет гарантии, что координаты `world_to_screen` вне `paint` совпадут с текущим
  кадром: используется последняя матрица.
- `rect_filled_blurred` зависит от цепочки blur xdraw; если она недоступна,
  вызов ничего не рисует.
- `push_font` вне `paint` глобально меняет шрифт xdraw. Это ответственность
  скрипта.
- Текст и изображения не обрезаются автоматически по границам экрана; для
  этого используй clip.
