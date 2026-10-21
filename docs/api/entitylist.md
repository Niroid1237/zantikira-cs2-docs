# entitylist

`z.entitylist` отдаёт **снимки** игрового мира. Это обычные Lua-таблицы, а не
сырые указатели на сущности: их можно сохранить, передать в другую функцию или
прочитать несколько раз. Данные в таблице не обновляются сами — каждый вызов
создаёт новый снимок.

Сырые pawn/controller-указатели, handle-ы и адреса через этот API не отдаются.

## Игроки

### z.entitylist.players()

```lua
local players = z.entitylist.players()
for _, player in ipairs(players) do
    z.log(player.index, player.name, player.health, player.local)
end
```

Возвращает массив **живых** игроков, отсортированный по `index` controller-
сущности. Пустой массив, если игроков нет. Локальный игрок входит в список и
отмечен `local = true`.

Живым считается controller с `m_bPawnIsAlive` и pawn с `health > 0`. Мёртвые
игроки и наблюдаемый pawn в observer-режиме в `players()` не попадают; для
наблюдаемого используй `local_player()`.

`index` — индекс controller-сущности, а не pawn и не номер слота в scoreboard.

### z.entitylist.local_player()

```lua
local me = z.entitylist.local_player()
if me and me.alive then
    z.log(me.name, me.health, me.weapon)
end
```

Возвращает снимок или `nil`, если controller/pawn ещё недоступны. Если игрок
мёртв и находится в observer-режиме, таблица описывает наблюдаемый pawn, а
`observer = true`.

Снимок `local_player()` использует те же поля, что и элементы `players()`, но
без `index`, `local`, `is_player`, `type` и `schema`.

### Поля снимка игрока

| Field | Type | Description |
|---|---|---|
| `index` | integer | controller index; только у `players()` и `entity()` |
| `local` | boolean | это локальный игрок; только у `players()` и `entity()` |
| `alive` | boolean | игрок жив |
| `observer` | boolean | активен observer-режим |
| `scoped` | boolean | оружие в режиме прицеливания |
| `health` | integer | здоровье; `0`, если недоступно |
| `team` | integer | номер команды; `0`, если неизвестно |
| `armor` | integer | броня; `0`, если недоступно |
| `ping` | integer | ping из controller; `0`, если недоступно |
| `name` | string или nil | санитизированное имя |
| `weapon` | string или nil | активное оружие без префикса `weapon_` |
| `weapon_id` | integer | item definition index; `0`, если неизвестно |
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

Векторы имеют формат `{ x, y, z }` и индексы `[1]`, `[2]`, `[3]`. Поля
`origin`, `eye_position`, `velocity`, `abs_velocity`, `name`, `weapon` и
`sim_time` могут отсутствовать — проверяй их на `nil`.

## Не-игровые сущности

### z.entitylist.all([include_players])

```lua
local entities = z.entitylist.all()
for _, entity in ipairs(entities) do
    z.log(entity.index, entity.type, entity.schema)
end
```

Возвращает все закэшированные не-игровые сущности: оружие, предметы, гранаты,
снаряды, двери и прочие объекты со схемой. По умолчанию player controller и
player pawn исключены.

`include_players = true` добавляет живых игроков **после** остальных записей, в
порядке controller index. Для чистого списка игроков используй `players()`.

`z.entitylist.entities()` — алиас `all()`.

### z.entitylist.items()

```lua
for _, item in ipairs(z.entitylist.items()) do
    z.log(item.index, item.schema)
end
```

Только сущности типа `item`: мировое оружие, ножи, гранаты в инвентаре и
предметы. Игроки и снаряды исключены.

### z.entitylist.projectiles()

Только летящие снаряды: гранаты, `C_Inferno` и другие projectile-сущности.

### z.entitylist.others()

Всё, что не попало в `item`, `projectile` и `player`: двери, world props и
прочие объекты со схемой.

### z.entitylist.entity(index)

```lua
local entity = z.entitylist.entity(42)
if entity then
    z.log(entity.type, entity.schema, entity.origin)
end
```

Ищет закэшированную сущность по `index`. Для игрока возвращает снимок игрока,
для остальных — общий снимок сущности. Если индекс не найден — `nil`.

`z.entitylist.get()` — алиас `entity()`.

### Поля снимка сущности

| Field | Type | Description |
|---|---|---|
| `index` | integer | индекс сущности |
| `type` | string | `item`, `projectile`, `other` или `player` |
| `type_id` | integer | числовой id типа |
| `is_player` | boolean | это player controller |
| `schema` | string или nil | имя схемы сущности |
| `pawn_schema` | string или nil | схема pawn, если известна |
| `health` | integer | `m_iHealth`; `0`, если недоступно |
| `alive` | boolean | `health > 0` |
| `team` | integer | команда; `0`, если неизвестно |
| `flags` | integer | `m_fFlags` |
| `on_ground` | boolean | `FL_ONGROUND` |
| `ducking` | boolean | `FL_DUCKING` |
| `dormant` | boolean или nil | `m_bDormant`, если scene node доступен |
| `origin` | vector или nil | позиция pawn/сущности |
| `eye_position` | vector или nil | позиция глаз, если применимо |
| `velocity` | vector или nil | скорость |
| `abs_velocity` | vector или nil | абсолютная скорость |
| `sim_time` | number или nil | simulation time |

У не-игровых сущностей нет `name`, `weapon`, `ammo` и `ping`. Это поля
player-снимка, а не общие свойства любой сущности.

Списки создаются из кэша `systems::g_entities` и сортируются по `index`.
Кэш обновляется движком чита; API не хранит ссылки на его элементы.

## История движения

### z.entitylist.history(index, limit?)

```lua
local samples = z.entitylist.history(player.index, 16)
for _, sample in ipairs(samples) do
    z.log(sample.tick, sample.sim_time, sample.origin)
end
```

Возвращает motion-снимки одного живого игрока от старых к новым. `index` —
controller index из `players()`. `limit` — от `1` до `64`, по умолчанию `16`.
Если истории нет, возвращается пустая таблица.

История собирается автоматически раз в `create_move`, пока включён scripting.
В неё попадают только живые игроки. Снимки живут в памяти до выключения,
перезагрузки чита или смены локального/наблюдаемого pawn. На диск ничего не
пишется.

| Field | Type | Description |
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
текущего состояния используй `players()`, для lag compensation — `z.combat`.

## Кости

### z.entitylist.bones(index)

```lua
for _, bone in ipairs(z.entitylist.bones(player.index)) do
    z.log(bone.id, bone.name, bone.position)
end
```

Возвращает массив всех доступных костей сущности. `index` — controller index
живого игрока, индекс живого player-pawn или индекс не-игровой сущности.
Элементы идут по bone id: `bone.id` совпадает с позицией в массиве (с нуля),
поэтому массив начинается с `[1]`, а `id` первого элемента — `0`.

Кость:

| Field | Type | Description |
|---|---|---|
| `id` | integer | bone id |
| `name` | string или nil | известное имя (`head`, `neck`, `spine_1`, ...) |
| `position` | vector | позиция в мире |
| `scale` | number | scale кости |
| `rotation` | quaternion | `{ x, y, z, w }` |

Для неизвестного id поле `name` отсутствует. Кости читаются через безопасные
чтения; при недоступной модели возвращается пустая таблица.

### z.entitylist.bone(index[, bone])

Возвращает одну кость. `bone` — числовой id `0..65535` или известное имя:
`pelvis`, `spine_1..spine_4`, `neck`, `head`, `left_clavicle`,
`left_shoulder`, `left_elbow`, `left_hand`, `right_clavicle`,
`right_shoulder`, `right_elbow`, `right_hand`, `left_hip`, `left_knee`,
`left_foot`, `right_hip`, `right_knee`, `right_foot`.

По умолчанию `head`. Если сущность не найдена — `nil`. Если id вне диапазона
или имя неизвестно — ошибка Lua.

### z.entitylist.bone_count(index)

Возвращает число доступных костей или `0`.

## Хитбоксы

### z.entitylist.hitboxes(index)

```lua
for _, hitbox in ipairs(z.entitylist.hitboxes(player.index)) do
    z.log(hitbox.index, hitbox.hitgroup_name, hitbox.origin)
end
```

Возвращает массив хитбоксов модели. Это данные hitbox set, а не готовые
попадания trace.

| Field | Type | Description |
|---|---|---|
| `index` | integer | индекс хитбокса в наборе |
| `bone` | integer | bone id, к которому привязан хитбокс |
| `hitgroup` | integer | hitgroup |
| `hitgroup_name` | string | имя hitgroup |
| `mins` | vector | локальные mins OBB/капсулы |
| `maxs` | vector | локальные maxs |
| `radius` | number | радиус капсулы |
| `shape` | integer | shape type из hitbox set |
| `translation_only` | boolean | флаг translation-only |
| `origin` | vector или nil | центр хитбокса в мире |

`origin` вычисляется как центр локального OBB, повёрнутый кватернионом кости и
сложенный с её позицией. Для капсулы это локальный центр между `mins` и
`maxs`; радиус отдельно. Углы OBB и точная геометрия капсулы не возвращаются.

### z.entitylist.hitbox(index, hitbox_index)

Возвращает один хитбокс или `nil`. Поля те же, кроме `shape` и
`translation_only` — одиночный вариант их не отдаёт.

## Видимость

### z.entitylist.visible(index[, bone])

```lua
local tr = z.entitylist.visible(player.index, "head")
```

Проверяет видимость точки кости из глаза локального/наблюдаемого pawn. По
умолчанию используется `head`. Возвращает `boolean`, а не таблицу trace.

- Если кость не найдена, используется `eye_position`, затем `origin`.
- Если trace-менеджер или глаз камеры ещё не готовы, возвращается `false`.
- В observer-режиме старт — глаз наблюдаемого pawn.
- `z.entitylist.can_see()` — алиас.

### z.entitylist.visible_hitbox(index, hitbox_index)

Проверяет видимость **центра** конкретного хитбокса. Это не проверка всех
углов OBB; для полной проверки используй собственные точки и `z.trace.line`.

## Удобные функции

```lua
z.entitylist.get_local_player()      -- local_player()
z.entitylist.get_local_player_pawn() -- local_player()
z.entitylist.get_local_player_name() -- string или nil
z.entitylist.get_local_weapon()      -- table или nil
z.entitylist.get_local_weapon_name() -- string или nil
z.entitylist.get_local_weapon_index()-- integer или nil
```

`get_local_weapon()` возвращает:

```lua
{
    name = "ak47",
    id = 7,
    ammo = 30,
    max_ammo = 30
}
```

Если активного оружия нет, возвращается `nil`.

## Пример: ESP-подобный список

```lua
z.events.paint:add(function()
    for _, player in ipairs(z.entitylist.players()) do
        if not player.local and player.origin then
            local x, y = z.render.world_to_screen(player.origin)
            if x and y then
                local color = { r = 255, g = 80, b = 80, a = 255 }
                z.render.text(x, y - 18, player.name or "unknown", color)
                z.render.text(x, y, ("%d hp"):format(player.health), color)
            end
        end
    end
end)
```

## Ограничения

- `players()` не возвращает мёртвых и наблюдаемого игрока.
- `all()` не сортирует игроков вместе с остальными: они добавляются в конец.
- `history()` ограничена `64` последними тиками и текущим pawn.
- Кости и хитбоксы могут быть недоступны для dormant/неигровых сущностей.
- `visible()` проверяет одну точку; это не autowall и не проверка всего тела.
- Снимки не обновляются. Не кэшируй их между тиками, если нужна свежая позиция.
