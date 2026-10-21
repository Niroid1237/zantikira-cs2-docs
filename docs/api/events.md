# События

Все события доступны через `z.events`:

```lua
z.events.create_move:add(function(cmd) end)
z.events.paint:add(function() end)
z.events.ui:add(function() end)
z.events.shutdown:add(function() end)
```

`add` — единственный метод события. Он принимает Lua-функцию и добавляет её в
конец списка. Один и тот же обработчик можно добавить несколько раз. Порядок
вызова — порядок добавления.

## create_move

| | |
|---|---|
| Аргументы | `cmd` — временный userdata |
| Когда | после встроенного movement, до финального subtick-обнуления |
| Что можно | `cmd:*`, `z.log`, `z.time`, `z.engine`, `z.entitylist`, `z.trace` |
| Что нельзя | `z.render` |

```lua
z.events.create_move:add(function(cmd)
    cmd:set_button(z.buttons.jump, true)
end)
```

`cmd` действителен только до возврата из обработчика. Если сохранить его в
глобальную переменную и использовать в следующем тике, методы выбросят ошибку
`command is only valid while create_move is running`. Это защищает от записи в
уже освобождённый usercmd.

Если обработчик вызывает `set_forward_move`, `set_left_move` или `set_up_move`,
менеджер помечает тик как script-controlled. Встроенное обнуление base movement
в этом тике пропускается. Установка кнопок и view angles такого флага не даёт.

## paint

| | |
|---|---|
| Аргументы | нет |
| Когда | после встроенных оверлеев, до виджетов и меню |
| Что можно | `z.render`, `z.log`, `z.time`, `z.engine`, `z.entitylist`, `z.trace` |
| Что нельзя | `cmd:*` |

```lua
z.events.paint:add(function()
    z.render.line(0, 0, 100, 100, { r = 255, g = 80, b = 80, a = 255 })
end)
```

`paint` вызывается и при закрытом меню. Рендер разрешён только внутри этого
события: вне его `z.render` выбрасывает ошибку.

## ui

| | |
|---|---|
| Аргументы | нет |
| Когда | внутри существующей пары `xui::begin`/`xui::end`, после окна меню |
| Что можно | `z.ui`, `z.settings`, `z.log`, `z.time`, `z.engine`, `z.entitylist`, `z.trace` |
| Что нельзя | `cmd:*`, `z.render` |

```lua
z.events.ui:add(function()
    z.ui.begin_window("my panel", 40, 40, 320, 200, true)
    z.ui.text("hello")
    z.ui.end_window()
end)
```

`ui` вызывается и при закрытом меню, поэтому окна скрипта остаются на экране.
Окна и виджеты используют текущий `xui`-контекст: тему, clipping, курсор и
очередь ввода. В отличие от `paint`, здесь нельзя рисовать через `z.render`.

Подробности: [пользовательские окна](ui.md) и [настройки](settings.md).

## shutdown

| | |
|---|---|
| Аргументы | нет |
| Когда | перед закрытием state при reload/выгрузке/смене конфига |
| Что можно | `z.log`, `z.time` |
| Что нельзя | `z.render`, `cmd:*` |

```lua
z.events.shutdown:add(function()
    z.log("bye")
end)
```

`shutdown` не вызывается после runtime-ошибки: в этом случае state закрывается
сразу, чтобы не выполнять код упавшего скрипта повторно.

## Ошибки и изоляция

- Ошибка в любом обработчике закрывает state этого скрипта.
- Другие скрипты продолжают работу.
- Ошибка пишется в лог ZantiKira и во вкладку Lua.
- Стек вызовов не прикладывается: сообщение Lua содержит источник и строку.
- Yield из `create_move`/`paint`/`ui` невозможен: вызов идёт через `lua_pcall`
  из C++.
- Окно, оставленное открытым в `ui`, принудительно закрывается после callback
  с предупреждением в логе.

## Порядок в кадре

```text
create_move:
  встроенный movement
  → Lua create_move
  → финальное subtick-обнуление
  → input apply
```

```text
frame:
  встроенные оверлеи
  → Lua paint
  → widgets / меню
  → Lua ui (окна скриптов поверх меню)
  → xui end / overlays
```
