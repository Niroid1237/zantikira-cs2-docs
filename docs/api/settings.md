# Настройки чита

`z.settings` даёт доступ к реестру настроек проекта по паре
`category, name`. Это тот же реестр, который сохраняется в конфиг и который
показывает меню; скрипт не получает raw-указатели.

## Поиск

```lua
local all = z.settings.list()
local ragebot = z.settings.list("ragebot - rifle")

for _, entry in ipairs(ragebot) do
    z.log(("%s.%s = %s"):format(entry.category, entry.name, entry.type))
end
```

`list([category])` возвращает массив таблиц:

```lua
{
    category = "ragebot - rifle",
    name = "hit chance",
    type = "int", -- bool, int, float, color, vec3, bool[], string, custom
}
```

`category` и `name` чувствительны к регистру и пробелам. Если не знаешь точное
имя, начни с `list()`; custom-поля (списки скинов, агенты, список скриптов)
не читаются и не пишутся как примитивы.

## Чтение и запись

```lua
z.settings.exists("ragebot - rifle", "hit chance") -- boolean
z.settings.type("ragebot - rifle", "hit chance")   -- "int" или nil

local value = z.settings.get("ragebot - rifle", "hit chance")
z.settings.set("ragebot - rifle", "hit chance", 65)
```

Поддерживается и короткая форма с точкой:

```lua
local value = z.settings.get("ragebot - rifle.hit chance")
z.settings.set("ragebot - rifle.hit chance", 65)
```

`get` возвращает значение в Lua-типе:

| `type` | Lua |
|---|---|
| `bool` | boolean |
| `int` | integer |
| `float` | number |
| `color` | `{ r, g, b, a }` и `[1]`…`[4]` |
| `vec3` | `{ x, y, z }` и `[1]`…`[3]` |
| `bool[]` | массив boolean |
| `string` | string |
| `custom` | ошибка |

`set` принимает значение того же типа и возвращает `true`. Для `xui::setting`
одновременно обновляется `value` и состояние toggle-bind, чтобы меню и
горячая клавиша не расходились. Для bind-режимов `hold on`/`hold off`
следующий кадр может перезаписать значение по клавише — это ожидаемое
поведение.

Ошибки:

- неизвестная пара `category, name` — ошибка Lua;
- неверный тип значения — ошибка Lua;
- попытка читать/писать `custom` как примитив — ошибка Lua;
- строка длиннее 4096 байт — ошибка Lua.

## Пример

```lua
local enabled = true

z.events.ui:add(function()
    z.ui.begin_window("settings demo", 80, 80, 360, 180)

    enabled = z.ui.checkbox("show hit chance", enabled)

    if enabled then
        local current = z.settings.get("ragebot - rifle", "hit chance")
        local next_value = z.ui.slider_int("hit chance", current, 0, 100)
        if next_value ~= current then
            z.settings.set("ragebot - rifle", "hit chance", next_value)
        end
    end

    z.ui.end_window()
end)
```

## Границы

- `z.settings` даёт доступ ко всем primitive-полям, включая боевые и
  визуальные. Ошибка в скрипте может изменить поведение чита; проверяй
  значения и не зацикливай запись каждый кадр без необходимости.
- Нет отдельного allowlist настроек и нет отката после ошибки. Ответственность
  за изменения лежит на авторе скрипта.
- Запись идёт в текущий runtime-конфиг. Сохранение в файл — обычным способом
  через меню/конфиг, API не вызывает `save` автоматически.
