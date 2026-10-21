# settings

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

`category` и `name` чувствительны к регистру и пробелам. Фильтр сравнивает
категорию целиком. Если не знаешь точное имя, начни с `list()`.

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

В короткой форме путь делится по **первой** точке. `category` и `name` должны
быть длиной `1..120` байт, весь путь — `3..241` байт. Пара из двух аргументов
предпочтительнее: она не зависит от того, есть ли точка в названии категории.

`get` возвращает значение в Lua-типе:

| Type | Lua |
|---|---|
| `bool` | boolean |
| `int` | integer |
| `float` | number |
| `color` | `{ r, g, b, a }` и `[1]`…`[4]` |
| `vec3` | `{ x, y, z }` и `[1]`…`[3]` |
| `bool[]` | массив boolean |
| `string` | string |
| `custom` | table / array / scalar по JSON-схеме поля |

`set` принимает значение того же типа и возвращает `true`. Для `xui::setting`
одновременно обновляется `value` и состояние toggle-bind, чтобы меню и
горячая клавиша не расходились. Для bind-режимов `hold on`/`hold off`
следующий кадр может перезаписать значение по клавише — это ожидаемое
поведение.

Особенности примитивов:

- `uint8`-поля обрезаются до `0..255`.
- `float` и `vec3` требуют конечных чисел (`nan`/`inf` — ошибка).
- `bool[]` заполняется по количеству поля: отсутствующие элементы становятся
  `false`, лишние игнорируются.
- Строка не имеет искусственного лимита длины; ограничение — память процесса.
- `color` принимает таблицу или упакованное `0xAARRGGBB`.

Ошибки:

- неизвестная пара `category, name` — ошибка Lua;
- неверный тип значения — ошибка Lua;
- неверная длина пути — ошибка Lua;
- циклическая таблица в custom-значении — ошибка Lua;
- ключ custom-таблицы не строка и не число — ошибка Lua;
- функция, userdata или другой не-JSON тип в custom-значении — ошибка Lua.

## Custom-поля

Custom-поля читаются и пишутся как обычные Lua-таблицы, массивы и скаляры.
`get` возвращает полную JSON-проекцию поля, `set` **заменяет** её целиком, а не
сливает с текущей. Если нужно изменить одно вложение, сначала прочитай
таблицу, измени её и запиши обратно.

Правила конвертации:

- таблица `1..n` без дыр становится JSON-массивом;
- остальные таблицы становятся JSON-объектами;
- ключи объектов — строки или числа (числа превращаются в строки);
- `nil` становится `null`;
- вложенные таблицы поддерживаются, циклы запрещены;
- JSON-парсер и произвольные схемы не навязываются: структуру определяет само
  поле.

### changer.applied skins

Объект, ключ — definition index оружия строкой, значение:

```lua
{
    p = 0,    -- paint kit id
    w = 0.01, -- wear
    s = 0,    -- seed
    t = 0,    -- StatTrak count
    d = 0,    -- override definition index
}
```

```lua
local skins = z.settings.get("changer", "applied skins")
skins["7"] = { p = 524, w = 0.01, s = 1, t = 0, d = 0 }
z.settings.set("changer", "applied skins", skins)
```

Значения `p` должны быть корректными paint kit id; API не проверяет, существует
ли скин, и не подбирает его за тебя.

### changer.agents

```lua
{
    ct = 0, -- definition index агента CT
    t = 0,  -- definition index агента T
}
```

```lua
local agents = z.settings.get("changer", "agents")
agents.ct = 0
agents.t = 0
z.settings.set("changer", "agents", agents)
```

### changer.custom agents

```lua
{
    selected_ct = 0,
    selected_t = 0,
    entries = {
        {
            name = "My Agent",
            model_path = "agents/models/my_agent/my_agent.vmdl",
            team = 3, -- 2 = T, 3 = CT
        },
    },
}
```

Неизвестные ключи игнорируются, отсутствующие получают значения по умолчанию
(`name = "Custom Agent"`, пустой `model_path`, `team = 3`). `settings.set`
заменяет весь список записей.

### lua.script list

```lua
{
    { path = "bhop.lua", enabled = true },
    { path = "overlay.lua", enabled = false },
}
```

`set` для этого поля ставит перезагрузку менеджера в очередь. Она выполняется
на ближайшем `on_present` вне активного Lua-callback. Текущий скрипт может
исчезнуть из списка или быть выключен — код после `set` всё равно выполнится
до перезагрузки, но не рассчитывай на следующий кадр.

Запись в `lua.script list` не создаёт файл и не проверяет его существование;
ошибка появится в логе при загрузке.

### Пример: включить скрипт по имени

```lua
local list = z.settings.get("lua", "script list")
for _, entry in ipairs(list) do
    if entry.path == "bhop.lua" then
        entry.enabled = true
    end
end
z.settings.set("lua", "script list", list)
```

## Пример с UI

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

- `z.settings` даёт доступ ко всем полям реестра, включая боевые и визуальные.
  Ошибка в скрипте может изменить поведение чита; проверяй значения и не
  зацикливай запись каждый кадр без необходимости.
- Нет отдельного allowlist настроек и нет отката после ошибки. Ответственность
  за изменения лежит на авторе скрипта.
- Чтение custom-поля всегда отдаёт снимок. Изменение таблицы в Lua не меняет
  настройку, пока не вызван `set`.
- Нет API для создания новых полей, изменения метаданных или удаления
  настроек.
- Запись идёт в текущий runtime-конфиг. Сохранение в файл — обычным способом
  через меню/конфиг, API не вызывает `save` автоматически.
- Схемы `changer.*` могут расширяться. Если поле вернуло неожиданную форму,
  логируй и проверяй её, а не предполагай фиксированный набор ключей.
