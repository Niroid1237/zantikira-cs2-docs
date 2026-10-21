# combat

`z.combat` объединяет три вещи, которые нельзя сделать одним `z.trace`:
проверку пробития (autowall), доступ к валидным lag compensation-записям и
autowall по конкретной цели с учётом её записанных костей.

Все функции синхронные. Они используют текущее оружие локального игрока и уже
собранные данные `g_shared`; это не отдельный симулятор урона. Наружу отдаются
только Lua-таблицы, без pawn/controller-указателей и без указателей на записи.

## z.combat.autowall(start, end[, target])

```lua
local start = z.entitylist.local_player().eye_position
local target = z.entitylist.players()[1]

local result = z.combat.autowall(start, target.origin, target.index)
z.log(result.hit, result.damage, result.hitbox, result.hitgroup_name)
```

`start` и `end` — конечные векторы `{ x, y, z }`. `target` — необязательный
`index` из `z.entitylist.players()` (controller index) или индекс живого
player-pawn. Если `target` не указан, выполняется обычная проверка пробития в
направлении `end - start`.

### Без цели

Используется `g_shared.pen().can()`. Результат описывает только факт урона и
его величину; конкретная сущность, хитбокс и hitgroup не возвращаются.

| Field | Type | Description |
|---|---|---|
| `hit` | boolean | урон больше нуля |
| `damage` | number | оценка урона после пробития |
| `penetrated` | boolean | совпадает с `hit` |
| `target` | integer | `0` |

### С целью

Берётся самая новая валидная запись lag compensation для цели, вызывается
`prepare_target()`, затем `pen().run()`. Это единственный путь, который
учитывает записанные кости и хитбоксы, а не только текущую модель.

| Field | Type | Description |
|---|---|---|
| `hit` | boolean | `run()` вернул попадание |
| `damage` | number | урон по цели |
| `penetrated` | boolean | была ли пробита поверхность |
| `target` | integer | переданный `target` |
| `hitbox` | integer | индекс хитбокса или `-1` |
| `hitgroup` | integer | hitgroup или `-1` |
| `hitgroup_name` | string | имя hitgroup или `"none"` |
| `tick` | integer | tick использованной записи |

Если локальный игрок мёртв, цель не является живым игроком или для неё нет
валидных записей, возвращается таблица с `hit = false`, `damage = 0` и полем
`reason`. Это не ошибка Lua: скрипт может спокойно проверить `result.hit`.

### Ограничения

- `start` и `end` должны различаться; иначе функция выбрасывает ошибку.
- Цель должна быть живым игроком: controller index разрешается в pawn, но
  мёртвые controller/pawn отклоняются.
- Урон зависит от текущего оружия, брони, hitgroup и накопленного
  penetration-контекста. Вызов в неподходящий момент может дать `0`.
- Запись lag compensation — снимок. После вызова она не обновляется.
- Функция не применяет запись к игре и не двигает цель.

## z.combat.backtrack(index)

```lua
local record = z.combat.backtrack(player.index)
if record then
    z.log(record.tick, record.sim_time, record.bone_count)
end
```

Возвращает самую новую валидную запись lag compensation для игрока или `nil`,
если записей нет. `index` — controller index или индекс живого pawn.

| Field | Type | Description |
|---|---|---|
| `index` | integer | переданный индекс игрока |
| `tick` | integer | tick записи |
| `sim_time` | number | simulation time |
| `valid` | boolean | запись была валидной |
| `was_valid` | boolean | запись была валидной в предыдущем обновлении |
| `extrapolated` | boolean | запись экстраполирована |
| `flags` | integer | `m_fFlags` на момент записи |
| `origin` | vector | позиция pawn |
| `bone_count` | integer | число скопированных костей |
| `bones` | table | массив костей |

Каждая кость:

```lua
{
    id = 7,
    name = "head",       -- может отсутствовать для неизвестного id
    position = { x = 0, y = 0, z = 0 },
    scale = 1.0,
    rotation = { x = 0, y = 0, z = 0, w = 1 },
}
```

`bones` — копия до `128` костей из записи. Это не ссылка на игру: таблицу можно
хранить, но она не обновится.

## z.combat.backtrack_records(index, limit?)

```lua
local records = z.combat.backtrack_records(player.index, 16)
for _, record in ipairs(records) do
    z.log(record.tick, record.origin)
end
```

Возвращает валидные записи от старых к новым. `limit` по умолчанию `16`; если
записей меньше, возвращаются все. Отрицательный или нулевой `limit`
поднимается до `1`. Формат элементов совпадает с `backtrack()`, включая поле
`index`.

`z.combat.latest_record()` — алиас `backtrack()`. `z.combat.can_shoot()` —
алиас `autowall()`.

## Пример: выбор цели по backtrack

```lua
z.events.create_move:add(function()
    local me = z.entitylist.local_player()
    if not me or not me.alive or not me.eye_position then
        return
    end

    for _, player in ipairs(z.entitylist.players()) do
        if not player.local then
            local record = z.combat.backtrack(player.index)
            if record then
                for _, bone in ipairs(record.bones) do
                    if bone.name == "head" then
                        local result = z.combat.autowall(me.eye_position, bone.position, player.index)
                        if result.hit and result.damage > 0 then
                            z.log(("hit %s for %d"):format(player.name or "?", result.damage))
                        end
                        break
                    end
                end
            end
        end
    end
end)
```

Autowall по каждой кости дорогой: engine trace и penetration нельзя вызывать
бесконечно на каждом тике. Ограничивай число целей и точек.

## Связанные модули

- [entitylist](entitylist.md) — кости, хитбоксы и проверка видимости.
- [trace](trace.md) — один луч/hull без penetration.
- [engine](engine.md) — prediction состояния локального игрока.
