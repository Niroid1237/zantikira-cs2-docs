# assets

`z.assets` загружает изображения, SVG и шрифты. Наружу отдаются userdata-токены
`image` и `font`, а не texture/SRV/font-указатели. Токены можно передавать в
`z.render` и в методы draw-list.

Загрузка выполняется синхронно. Большой файл или сложный SVG/шрифт на короткое
время блокирует поток, из которого вызван скрипт.

## Пути

- Относительный путь сначала проверяется от `%LOCALAPPDATA%\zantikira\scripts`,
  затем от текущей рабочей папки процесса.
- Абсолютный путь принимается как есть.
- В отличие от списка скриптов, пути ассетов не обязаны находиться внутри
  папки скриптов. Не запускай чужой код, который читает произвольные файлы.

## z.assets.load_image(path[, scale])

```lua
local image, err = z.assets.load_image("images/logo.png")
if not image then
    z.log.error(err)
    return
end

z.render.image(20, 20, 128, 128, image)
```

Алиас: `z.assets.image()`.

Для `.svg` второй аргумент задаёт масштаб (по умолчанию `1.0`). Для остальных
форматов он игнорируется. Поддерживаются растровые форматы xdraw, GIF и SVG.

Возвращает `image` или `nil, error`.

### image

| Method | Return | Description |
|---|---|---|
| `image:valid()` | boolean | текстура/кадр доступны |
| `image:size()` | width, height | размеры в пикселях |
| `image:frames()` | integer | число кадров; `1` для статичной картинки |

GIF-анимация обновляется при каждом использовании изображения в кадре.

## z.assets.load_svg(svg_text[, scale])

```lua
local icon = z.assets.load_svg([[
<svg xmlns="http://www.w3.org/2000/svg" width="32" height="32">
  <circle cx="16" cy="16" r="14" fill="#8b9cff"/>
</svg>
]], 1.0)
```

Загружает SVG из строки. Возвращает `image` или `nil, error`.

## z.assets.load_font(path, size[, atlas_w, atlas_h])

```lua
local font, err = z.assets.load_font("fonts/Inter.ttf", 16.0)
if not font then
    z.log.error(err)
    return
end

z.render.text(20, 20, "custom font", { r = 255, g = 255, b = 255, a = 255 }, font)
```

| Parameter | Default | Description |
|---|---|---|
| `path` | — | путь к файлу шрифта |
| `size` | — | размер в пикселях, больше нуля |
| `atlas_w` | `1024` | ширина атласа |
| `atlas_h` | `1024` | высота атласа |

Возвращает `font` или `nil, error`. Размер и размеры атласа должны быть
положительными.

### font

| Method | Return | Description |
|---|---|---|
| `font:measure(text)` | width, height | размеры строки этим шрифтом |
| `font:size()` | size, line_height | размер и высота строки |

## Рендер изображений

```lua
z.render.image(x, y, w, h, image[, tint[, radius]])
z.render.image_uv(x, y, w, h, image, u0, v0, u1, v1[, tint])
```

`tint` — обычный цвет `{ r, g, b, a }` или упакованное целое. `radius`
скругляет углы. `image_uv` принимает нормализованные UV в диапазоне `0..1`.

```lua
z.render.image_uv(20, 20, 64, 64, atlas, 0.0, 0.0, 0.5, 0.5)
```

## Шрифты в render

```lua
z.render.text(x, y, text, color[, font][, style])
```

`style` — `"normal"`, `"outlined"` или `"shadowed"`. Шрифт можно не передавать,
тогда используется встроенный шрифт xdraw.

```lua
z.render.text(20, 60, "outlined", color, font, "outlined")
z.render.measure_text("text", font)
```

Есть также глобальный стек шрифта:

```lua
z.render.push_font(font)
z.render.text(20, 90, "uses pushed font", color)
z.render.pop_font()
```

Стек глобальный для xdraw. Всегда вызывай `pop_font()` в том же обработчике,
иначе следующие draw-вызовы (включая меню) останутся с чужим шрифтом.

## Время жизни

- Токен `image`/`font` принадлежит Lua state. После reload загрузи ассет заново.
- Изображения удерживаются внутренним keepalive, чтобы отложенный draw-list не
  ссылался на освобождённую текстуру.
- Шрифты принадлежат xdraw. Не пытайся освободить их вручную.
- Draw-list копирует команды, но не копирует содержимое текстур: токен/ассет
  должен оставаться живым до отправки списка.

## Ошибки

При ошибке чтения или декодирования возвращается `nil` и строка ошибки. Это не
исключение Lua, поэтому проверяй первый результат. `z.render.image` с
недействительным токеном выбрасывает ошибку userdata.
