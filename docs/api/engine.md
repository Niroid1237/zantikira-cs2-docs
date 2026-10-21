# engine

Состояние движка, клиентские команды и engine prediction локального игрока.

## z.engine.is_connected()

```lua
local connected = z.engine.is_connected()
```

Возвращает `true`, если интерфейс клиента и `global_vars` уже инициализированы.
Это **не** проверка активного сетевого соединения и не гарантия, что игрок уже
на сервере. Для проверки локального pawn/observer используй
`z.engine.is_in_game()`.

## z.engine.is_in_game()

```lua
local in_game = z.engine.is_in_game()
```

Возвращает `true`, если существует локальный pawn или observer-pawn. То же
значение отдаёт `z.globals.is_in_game()`.

## z.engine.client_cmd(command)

```lua
z.engine.client_cmd("say hello")
```

Выполняет клиентскую команду через движок. Команда выполняется сразу, без
очереди и без возврата результата.

| Parameter | Constraint |
|---|---|
| `command` | непустая строка, без NUL-байтов |
| размер | искусственного лимита нет; ограничен памятью процесса |
| доступность | нужен инициализированный engine client |

При недоступном движке или паттерне функция выбрасывает ошибку Lua.

!!! warning "Произвольные команды"
    Это не безопасный API movement. Через `client_cmd` можно выполнить любую
    клиентскую команду, включая `quit`, `disconnect`, `bind` и команды,
    меняющие состояние игры. Скрипт полностью отвечает за то, что отправляет.
    Для движения используй `cmd:set_*`, а не клиентские команды.

`client_cmd` не предназначен для вызова каждый тик. Если команда включает
состояние (`+attack`), не забывай про парную команду (`-attack`).

## z.engine.prestate()

```lua
local state = z.engine.prestate()
```

Возвращает сохранённое prestate предыдущего `g_prediction.simulate()`:

| Field | Type | Description |
|---|---|---|
| `flags` | integer | `m_fFlags` до симуляции |
| `on_ground` | boolean | `FL_ONGROUND` |
| `ducking` | boolean | `FL_DUCKING` |
| `networked_velocity` | vector | `m_vecVelocity` |
| `velocity` | vector | `m_vecAbsVelocity` |
| `origin` | vector | позиция pawn |
| `networked_origin` | vector | сетевая позиция |
| `last_movement_impulses` | vector | movement impulses |
| `surface_friction` | number | surface friction |
| `stamina` | number | stamina movement services |

До первого `simulate()` таблица содержит значения по умолчанию. Это снимок, а
не живое состояние.

## z.engine.predict_state([cmd])

```lua
z.events.create_move:add(function(cmd)
    local predicted = z.engine.predict_state(cmd)
    z.log(predicted.origin, predicted.velocity, predicted.on_ground)
end)
```

Запускает `g_prediction.simulate()` с пустым callback, читает состояние
локального pawn до восстановления игры и возвращает таблицу:

| Field | Type | Description |
|---|---|---|
| `predicted` | boolean | симуляция выполнена |
| `health` | integer | здоровье |
| `team` | integer | команда |
| `flags` | integer | `m_fFlags` после симуляции |
| `on_ground` | boolean | `FL_ONGROUND` |
| `ducking` | boolean | `FL_DUCKING` |
| `origin` | vector или nil | предсказанная позиция |
| `eye_position` | vector или nil | предсказанная позиция глаз |
| `velocity` | vector или nil | `m_vecVelocity` |
| `abs_velocity` | vector или nil | `m_vecAbsVelocity` |
| `sim_time` | number или nil | simulation time |

Вне `create_move` передай `cmd` явно. Без аргумента и без текущего
`create_move` функция выбрасывает ошибку. Нужен живой локальный игрок.

## z.engine.predict(callback[, cmd])

```lua
z.events.create_move:add(function(cmd)
    z.engine.predict(function()
        -- игра уже переведена в предсказанное состояние
        local me = z.entitylist.local_player()
        z.log(me.origin)
    end, cmd)
end)
```

Выполняет Lua-callback внутри `g_prediction.simulate()`, пока игра находится в
предсказанном состоянии и до восстановления исходных данных. Это позволяет
прочитать предсказанную позицию, velocity, флаги и сделать trace от
предсказанной точки.

Алиас: `z.engine.simulate()`.

### Правила

- Нужен живой локальный игрок.
- Вне `create_move` нужен явный `cmd`; внутри `create_move` можно вызвать
  `z.engine.predict(fn)` и менеджер возьмёт текущий usercmd.
- Callback не должен вызывать `z.engine.predict`/`predict_state` повторно:
  вложенная prediction запрещена.
- Не вызывай `z.scripts.reload()` внутри callback prediction. Reload сам
  откладывается, но лучше делать это вне симуляции.
- Ошибка Lua в callback перехватывается, симуляция завершается и восстанавливает
  состояние, затем ошибка возвращается в скрипт.
- `cmd` живёт только до конца текущего `create_move`.
- Prediction не делает autowall. Для урона используй `z.combat`.

## Пример: проверка позиции после симуляции

```lua
z.events.create_move:add(function(cmd)
    local ok = z.engine.predict(function()
        local me = z.entitylist.local_player()
        if me and me.eye_position then
            local tr = z.trace.line(me.eye_position, {
                x = me.eye_position.x,
                y = me.eye_position.y,
                z = me.eye_position.z - 8192,
            })
            z.log(tr.fraction)
        end
    end)
end)
```

## Связанные модули

- [command](command.md) — изменение usercmd.
- [entitylist](entitylist.md) — снимки и кости.
- [combat](combat.md) — autowall и backtrack.
