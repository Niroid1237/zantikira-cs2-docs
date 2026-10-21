# entitylist

`z.entitylist` отдаёт **снимки** данных игроков. Это обычные Lua-таблицы, а не
сырые указатели на игровые сущности: их можно сохранить, передать в другую
функцию или прочитать несколько раз. Данные в таблице не обновляются сами —
каждый вызов создаёт новый снимок.

## z.entitylist.players()

```lua
local players = z.entitylist.players()
for _, player in ipairs(players) do
    z.log(player.index, player.name, player.health, player.velocity)
end
```

Возвращает массив живых игроков. Пустой массив, если игроков нет. Если
локальный игрок жив, он тоже входит в список и отмечен полем `local = true`.

Список отсортирован по `index` — индексу controller-сущности. Это не индекс
pawn и не номер слота в scoreboard. В таблице есть только значения, скопированные
из памяти; указатели и handle-ы наружу не отдаются.

Живым считается controller с `m_bPawnIsAlive` и pawn с `health > 0`. Мёртвые
игроки в список не попадают. Для наблюдаемого игрока в observer-режиме используй
`local_player()`: `players()` возвращает обычных живых игроков.

## z.entitylist.history(index, limit?)

```lua
local samples = z.entitylist.history(player.index, 16)
for _, sample in ipairs(samples) do
    z.log(sample.tick, sample.sim_time, sample.origin)
end
```

Возвращает массив motion-снимков одного игрока, от старых к новым. `index` —
controller index из `players()`, `limit` — от 1 до 64, по умолчанию 16. Если
истории нет, возвращается пустая таблица.

История собирается автоматически один раз за `create_move`, пока включён
scripting. В неё попадают только живые игроки. Снимки живут в памяти до
выключения/перезагрузки чита; на диск ничего не пишется. В меню и вне игры
новые снимки не появляются. При смене локального или наблюдаемого pawn история
сбрасывается, а снимки старого pawn с тем же index не возвращаются после
респавна.

| Поле | Тип | Описание |
|---|---|---|
| `index` | integer | controller index |
| `tick` | integer | tick/command number снимка; `0`, если недоступно |
| `time` | number | curtime снимка; `0`, если недоступно |
| `health` | integer | здоровье |
| `team` | integer | команда |
| `origin` | vector или nil | позиция pawn |
| `eye_position` | vector или nil | позиция глаз |
| `velocity` | vector или nil | `m_vecVelocity` |
| `abs_velocity` | vector или nil | `m_vecAbsVelocity` |
| `flags` | integer | `m_fFlags` |
| `on_ground` | boolean | `FL_ONGROUND` |
| `ducking` | boolean | `FL_DUCKING` |
| `sim_time` | number или nil | `m_flSimulationTime` |

В history нет имени, оружия, брони и ping: это лёгкие motion-снимки. Для
текущего состояния используй `players()`.

## z.entitylist.local_player()

```lua
local player = z.entitylist.local_player()
if player and player.alive then
    z.log(player.name, player.health, player.weapon)
end
```

Возвращает таблицу или `nil`, если локальный controller и pawn ещё недоступны.
Если игрок мёртв и находится в observer-режиме, таблица описывает наблюдаемого
pawn, а поле `observer` равно `true`.

## Поля снимка

| Поле | Тип | Описание |
|---|---|---|
| `index` | integer | индекс controller-сущности; только у `players()` |
| `local` | boolean | это локальный игрок; только у `players()` |
| `alive` | boolean | игрок жив; у `local_player()` отражает локального игрока |
| `observer` | boolean | активен observer-режим; у `players()` всегда `false` |
| `scoped` | boolean | оружие в режиме прицеливания |
| `health` | integer | здоровье; `0`, если недоступно |
| `team` | integer | номер команды; `0`, если неизвестно |
| `armor` | integer | броня; `0`, если недоступно |
| `ping` | integer | ping из controller; `0`, если недоступно |
| `name` | string или nil | санитизированное имя игрока |
| `weapon` | string или nil | имя активного оружия без префикса `weapon_` |
| `weapon_id` | integer | item definition index; `0`, если оружие неизвестно |
| `ammo` | integer | патроны в текущем магазине |
| `max_ammo` | integer | размер магазина; `0`, если недоступно |
| `origin` | vector или nil | абсолютная позиция pawn |
| `eye_position` | vector или nil | позиция глаз с учётом view offset |
| `velocity` | vector или nil | сетевая скорость `m_vecVelocity` |
| `abs_velocity` | vector или nil | абсолютная скорость `m_vecAbsVelocity` |
| `flags` | integer | сырые флаги `m_fFlags`; `0`, если недоступно |
| `on_ground` | boolean | флаг `FL_ONGROUND` |
| `ducking` | boolean | флаг `FL_DUCKING` |
| `sim_time` | number или nil | `m_flSimulationTime` в секундах |

Векторы имеют тот же формат, что и в `z.math`: `{ x, y, z }` и индексы
`[1]`, `[2]`, `[3]`. Поля `origin`, `eye_position`, `velocity`, `abs_velocity`
и `sim_time` могут отсутствовать, если соответствующие данные ещё не готовы.
Проверяй их на `nil` перед использованием.

`velocity` и `abs_velocity` — это скорость в игровых единицах в секунду.
`sim_time` удобно использовать для оценки интервала между снимками. `flags`
можно читать целиком, а `on_ground` и `ducking` уже вычислены для типичных
проверок.

## Своя экстраполяция

`players()` даёт данные, но не выполняет prediction за тебя. Простую линейную
оценку можно сделать в Lua:

```lua
local history = {}

z.events.create_move:add(function()
    for _, player in ipairs(z.entitylist.players()) do
        local prev = history[player.index]
        if prev and prev.sim_time and player.origin and player.velocity and player.sim_time
            and player.sim_time > prev.sim_time then
            local dt = player.sim_time - prev.sim_time
            if dt > 0 and dt <= 0.25 then
                local predicted = {
                    x = player.origin.x + player.velocity.x * dt,
                    y = player.origin.y + player.velocity.y * dt,
                    z = player.origin.z + player.velocity.z * dt,
                }
                -- predicted — только оценка; коллизии и оружие здесь не учитываются
            end
        end

        history[player.index] = {
            sim_time = player.sim_time,
        }
    end
end)
```

Для сложной экстраполяции нужны trace, netvars/cvars, кости и история чужих
игроков. Этого в `z.entitylist` пока нет: API не притворяется полноценным
engine prediction. FFI не даёт готовых оффсетов или масок; собственный native-код
на FFI — полностью твоя ответственность.

Имя в observer-режиме относится к наблюдаемому игроку. Не считай его именем
локального аккаунта.

## Алиасы

```lua
z.entitylist.get_local_player()      -- то же, что local_player()
z.entitylist.get_local_player_pawn() -- то же, что local_player()
```

`get_local_player_pawn()` возвращает такую же таблицу, а не pointer. Сырые
указатели через этот API не отдаются.

## Удобные функции

```lua
z.entitylist.get_local_player_name()  -- string или nil
z.entitylist.get_local_weapon()       -- table или nil
z.entitylist.get_local_weapon_name()  -- string или nil
z.entitylist.get_local_weapon_index() -- integer или nil
```

`get_local_weapon()` возвращает таблицу:

```lua
{
    name = "ak47",
    id = 7,
    ammo = 30,
    max_ammo = 30
}
```

Если активного оружия нет, возвращается `nil`. `get_local_weapon_index()`
возвращает `nil`, когда оружие неизвестно.

## Пример

```lua
z.events.paint:add(function()
    local player = z.entitylist.local_player()
    if not player or not player.alive then
        return
    end

    local text = ("%s | %d hp | %s"):format(
        player.name or "unknown",
        player.health,
        player.weapon or "none"
    )

    z.render.rect_filled(20, 20, 260, 34, {
        r = 12, g = 12, b = 18, a = 220
    }, 5)
    z.render.text(30, 29, text, {
        r = 235, g = 235, b = 245, a = 255
    })
end)
```

Чтение снимка не требует FFI. Если FFI включён, таблица всё равно остаётся
обычной копией данных; для более глубокого доступа к памяти используй
собственный FFI-код на свой риск.
