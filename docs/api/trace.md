# trace

`z.trace` выполняет один луч или hull через игровой trace-менеджер. Это
низкоуровневый запрос: API не делает autowall, не перебирает точки и не
возвращает raw-указатели. Результат — обычная таблица с геометрией и индексом
игрока, если луч попал в pawn.

## z.trace.line(start, end, options?)

```lua
local tr = z.trace.line({ x = 0, y = 0, z = 64 }, { x = 0, y = 0, z = 0 })
if tr.hit then
    z.log(tr.fraction, tr.hit_player, tr.hit_player_index)
end
```

Трассирует отрезок `start -> end`. `start` и `end` — векторы `{ x, y, z }` или
`{ [1], [2], [3] }`. Все компоненты должны быть конечными числами.

## z.trace.hull(start, end, mins, maxs, options?)

```lua
local tr = z.trace.hull(
    eye, target,
    { x = -16, y = -16, z = 0 },
    { x = 16, y = 16, z = 72 }
)
```

Трассирует hull с локальными `mins`/`maxs`. Это удобно для проверки габаритов
игрока или препятствия, но не заменяет полноценный autowall.

## Опции

| Поле | Тип | По умолчанию | Описание |
|---|---|---|---|
| `skip_local` | boolean | `true` | не трассировать в локального/наблюдаемого pawn |
| `mask` | integer | `0x1c3003` | маска contents для trace |
| `layer` | integer | `4` | collision layer |

`skip_local` использует `view_pawn()`: в observer-режиме пропускается
наблюдаемый pawn. Если нужен «сырой» trace без пропуска — `skip_local = false`.

`mask` и `layer` — advanced-настройки. Меняй их только если понимаешь, какие
contents и layer нужны; стандартные значения подходят для обычной проверки
видимости и геометрии.

## Результат

| Поле | Тип | Описание |
|---|---|---|
| `hit` | boolean | луч во что-то попал |
| `fraction` | number | доля отрезка до попадания, `0..1` |
| `position` | vector | точка попадания |
| `normal` | vector | нормаль поверхности |
| `contents` | integer | contents hit-объекта |
| `all_solid` | boolean | стартовая точка полностью в solid |
| `hit_player` | boolean | попадание в pawn игрока |
| `hit_player_index` | integer или nil | controller index игрока, если `hit_player` |

`hit_player_index` — тот же controller index, что и `index` в
`z.entitylist.players()`. Сырые pawn/controller-указатели не отдаются. Если
`hit = true`, а `hit_player = false`, значит луч попал в мир или не-игровую
сущность.

## Пример проверки видимости

```lua
z.events.create_move:add(function()
    local me = z.entitylist.local_player()
    if not me or not me.alive or not me.eye_position then
        return
    end

    for _, player in ipairs(z.entitylist.players()) do
        if not player.local and player.eye_position then
            local tr = z.trace.line(me.eye_position, player.eye_position)
            if not tr.hit or tr.hit_player_index == player.index then
                -- цель не закрыта миром
            end
        end
    end
end)
```

Один вызов `z.trace.line` — один луч. Если нужна видимость по нескольким
точкам, вызывай trace несколько раз и ограничивай количество: engine trace
дороже, чем чтение snapshot из `z.entitylist`.

## Ограничения

- Нет автоматического penetration/autowall: один вызов не пробивает стены.
- Нет истории и backtrack: для этого используй `z.entitylist.history()` вместе
  со своей моделью.
- Нет доступа к hull конкретной модели оружия или костям.
- Trace доступен без FFI; FFI не нужен и не даёт готовых масок/оффсетов.
- Результат — копия данных. Он не обновляется и не хранит ссылку на сущность.
