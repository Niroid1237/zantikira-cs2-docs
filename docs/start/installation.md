# Установка

## Требования

- Загруженный `debug.dll`, `alpha.dll` или `release.dll`.
- Включённый чекбокс `enable scripting` во вкладке **Lua**.
- Файл `.lua` в папке скриптов ZantiKira.

## Папка скриптов

Основной путь:

```text
%LOCALAPPDATA%\zantikira\scripts
```

Если `LOCALAPPDATA` недоступна, используется
`%APPDATA%\phantom\scripts`, затем `.\scripts` рядом с процессом.

Относительный путь в конфиге разрешается от этой папки:

```json
{ "path": "my_script.lua", "enabled": true }
```

Абсолютный путь разрешён только внутри папки скриптов. Выход через `..`,
junction/symlink или другой корень отклоняется. Поддерживаются только `.lua`;
файл должен существовать и быть непустым. Искусственного лимита размера,
длины пути или числа записей больше нет.

Узнать точный путь из Lua:

```lua
z.log(z.scripts.directory())
z.scripts.open_directory()
```

## Список скриптов

Скрипты **не обнаруживаются автоматически**. Менеджер читает только
`lua.script list` из активного конфига — custom-поле с массивом записей:

```json
[
  { "path": "bhop.lua", "enabled": true },
  { "path": "overlay.lua", "enabled": false }
]
```

В JSON-файле конфига это поле лежит внутри `fields` под ключом `7605c6c2`.
Число записей не ограничено.

Добавлять записи можно через вкладку **Lua**:

1. Введи имя файла или путь в поле под `Scripts`.
2. Нажми `add`.
3. Нажми `reload` — только после этого скрипт запустится.
4. Для сохранения списка сохрани конфиг в **Settings → Config**.

`enabled` выключает отдельный скрипт. Глобальный `enable scripting` выключает
весь рантайм и выгружает все state.

Список можно менять из Lua, в том числе из самого скрипта:

```lua
local list = z.settings.get("lua", "script list")
table.insert(list, { path = "new_script.lua", enabled = true })
z.settings.set("lua", "script list", list) -- перезагрузка будет запрошена сама
```

Текущий скрипт может выключить сам себя. Код после `set` успеет выполниться, но
на следующем `on_present` все state будут перезагружены по новому списку.

## FFI

`allow ffi (unsafe)` — глобальный флаг активного конфига, по умолчанию
выключен. Он сохраняется вместе с конфигом. При переключении все Lua state
закрываются и создаются заново.

- **Выключен:** нет `io`, `os`, `debug`, `package`, `dofile`, `loadfile`, `ffi`;
  `require` всегда выбрасывает ошибку. Сеть и чтение ассетов при этом доступны
  через `z.net` и `z.assets`.
- **Включён:** полные стандартные библиотеки LuaJIT, включая `ffi` и `package`;
  `package.path` начинается с папки скриптов.

Подробнее: [безопасность](../security.md).

## Перезагрузка

Скрипты перезагружаются:

- кнопкой `reload` во вкладке Lua;
- из Lua через `z.scripts.reload()`;
- при изменении `lua.script list` через `z.settings.set`;
- при добавлении/удалении/выключении записи в списке;
- при смене `enable scripting` или `allow ffi`;
- при загрузке или сбросе конфига.

`z.scripts.reload()` только ставит перезагрузку в очередь: она выполняется на
ближайшей безопасной границе, вне активного Lua-callback. При перезагрузке
вызывается `shutdown`, затем state закрывается. Если скрипт падает во время
`create_move` или `paint`, закрывается только его state; ошибка попадает в лог,
остальные скрипты продолжают работу.

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
| `socket is closed` | метод сокета после `close()` |
| `predict must be called from create_move or with a command` | нет usercmd и не передан явный `cmd` |
