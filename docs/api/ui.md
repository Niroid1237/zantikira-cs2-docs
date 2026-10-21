# Пользовательские окна

`z.ui` — immediate-mode API в стиле ImGui поверх существующего `xui`. Это не
второй Dear ImGui context: окна используют текущий рендер, тему, clipping и
очередь ввода меню. Поэтому `z.ui` доступен только внутри события
[`z.events.ui`](events.md), а не в `paint`.

## Окно

```lua
z.events.ui:add(function()
    local sw, sh = z.ui.screen_size()
    z.ui.begin_window("my panel", sw - 360, 80, 320, 260, true)
    z.ui.text("hello")
    z.ui.end_window()
end)
```

```lua
z.ui.begin_window(title, x, y, w, h[, resizable])
z.ui.end_window()
z.ui.set_window_rect(title, x, y, w, h)
```

Позиция и размер сохраняются в состоянии скрипта: при следующем кадре окно
остаётся там, куда его перетащили. `set_window_rect` меняет сохранённый
прямоугольник (например, для сброса раскладки). Заголовок должен быть
уникальным в пределах скрипта. Одно окно на один обработчик `ui`: окно нужно
закрыть до возврата из callback; незакрытое окно закроется принудительно с
предупреждением в логе.

`resizable` включает ручку изменения размера в правом нижнем углу. Окно
перетаскивается за заголовок. Минимальный размер — 160×100.

## Виджеты

```lua
z.ui.text(text[, color])
z.ui.separator()
z.ui.same_line([offset])
z.ui.spacing([amount])

local pressed = z.ui.button(label[, width, height])
local value   = z.ui.checkbox(label, value)
local value   = z.ui.slider(label, value, min, max[, format])
local value   = z.ui.slider_int(label, value, min, max[, format])
local index   = z.ui.combo(label, index, items[, width])
local text    = z.ui.text_input(label, value[, max_len[, hint]])

local w, h = z.ui.screen_size()
local w, h = z.ui.measure_text(text)
```

Цвет — таблица `{ r, g, b, a }` или упакованное `0xAARRGGBB`, как в
`z.render`.

`combo` принимает массив строк и индекс, начиная с `1`. Выбранный индекс
хранится в состоянии скрипта; второй аргумент задаёт начальное значение.
`text_input` возвращает текущий буфер; буфер сохраняется между кадрами, пока
поле активно.

`ui` вызывается и при закрытом меню, поэтому окно остаётся на экране. Но пока
игра держит курсор захваченным, клики и ввод в него недоступны: интерактивные
панели рассчитаны на открытое меню, а визуальные оверлеи можно показывать
всегда.

Суффикс `##id` скрывает часть подписи и делает ID уникальным:

```lua
z.ui.slider("speed##main", speed, 0.1, 10.0)
z.ui.slider("speed##alt", speed, 0.1, 10.0)
```

## Пример панели

```lua
local enabled = true
local speed = 1.0
local mode = 1
local name = "player"

z.events.ui:add(function()
    local sw, sh = z.ui.screen_size()
    z.ui.begin_window("demo", sw - 360, 80, 320, 300, true)

    z.ui.text("zantikira lua", { r = 139, g = 156, b = 255, a = 255 })
    z.ui.separator()

    enabled = z.ui.checkbox("enabled", enabled)
    speed = z.ui.slider("speed", speed, 0.1, 10.0, "%.2f")
    mode = z.ui.combo("mode", mode, { "off", "legacy", "strict" }, 160)
    name = z.ui.text_input("name", name, 64, "player name")

    if z.ui.button("apply", 100, 24) then
        z.log(("enabled=%s speed=%.2f mode=%d name=%s"):format(
            tostring(enabled), speed, mode, name))
    end

    z.ui.end_window()
end)
```

## Ограничения

- Окно и все виджеты должны вызываться между `begin_window` и `end_window`.
- `z.ui` нельзя вызывать из `create_move`, `paint`, `shutdown` или загрузки
  скрипта.
- Пока поддерживаются текстовые поля, checkbox, slider, combo, кнопка,
  текст, разделитель, `same_line` и `spacing`. Деревья, таблицы, color picker
  и keybind-виджеты появятся отдельно.
- Значения виджетов не привязаны к настройкам автоматически; для этого
  используй [`z.settings`](settings.md).
