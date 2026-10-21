# ZantiKira Lua API

**Версия:** `0.7.0`  
**Runtime:** LuaJIT 2.1, один `lua_State` на скрипт  
**Публичная документация:** <https://niroid1237.github.io/zantikira-cs2-docs/>

Скрипты запускаются только из явного списка `lua.script list` в активном
конфиге. Папка не сканируется, облака и allowlist нет. FFI по умолчанию
выключен.

## Оглавление

- [Запуск](#запуск)
- [События](#события)
- [cmd](#cmd)
- [Глобальные таблицы](#глобальные-таблицы)
- [z.math](#zmath)
- [z.entitylist](#zentitylist)
- [z.combat](#zcombat)
- [z.trace](#ztrace)
- [z.engine](#zengine)
- [z.render](#zrender)
- [z.assets](#zassets)
- [z.net](#znet)
- [z.settings](#zsettings)
- [z.scripts](#zscripts)
- [z.ui](#zui)
- [FFI](#ffi)
- [Ограничения](#ограничения)
- [Примеры](#примеры)

## Запуск

```text
%LOCALAPPDATA%\zantikira\scripts
```

Fallback: `%APPDATA%\phantom\scripts`, затем `.\scripts` рядом с процессом.
Относительный путь разрешается от папки скриптов; абсолютный путь должен
оставаться внутри неё. Поддерживаются только `.lua`, файл должен быть непустым.

```lua
z.log("hello from " .. z.api.script)
```

Список хранится в `lua.script list`:

```json
[
  { "path": "bhop.lua", "enabled": true },
  { "path": "overlay.lua", "enabled": false }
]
```

`enable scripting` включает рантайм, `reload` перезагружает список. Добавление,
удаление и переключение записей, а также запись списка через `z.settings`
вызывают перезагрузку на ближайшей безопасной границе. Искусственных лимитов
на размер файла, длину пути, число записей и размер лога больше нет.

## События

```lua
z.events.create_move:add(function(cmd) end)
z.events.paint:add(function() end)
z.events.ui:add(function() end)
z.events.shutdown:add(function() end)
```

| Event | Context | cmd | Render | UI |
|---|---|---|---|---|
| `create_move` | после base movement, до subtick-обнуления | да, только внутри callback | отложенный | нет |
| `paint` | после встроенных оверлеев, до меню | нет | прямой | нет |
| `ui` | внутри `xui::begin`/`end` | нет | отложенный | да |
| `shutdown` | перед закрытием state | нет | не рекомендуется | нет |

`add` добавляет обработчик в конец списка. Ошибка закрывает state только
упавшего скрипта. `cmd` действителен только до возврата из `create_move`.
`z.scripts.reload()` не прерывает текущий callback.

## cmd

| Method | Return | Description |
|---|---|---|
| `cmd:get_button(mask)` | boolean | установлен ли бит |
| `cmd:set_button(mask, pressed)` | nil | поставить/снять бит |
| `cmd:get_buttons()` | integer | весь bitmask |
| `cmd:set_buttons(bits)` | nil | заменить bitmask |
| `cmd:get_view_angles()` | angle или nil | текущие view angles |
| `cmd:set_view_angles(pitch, yaw, roll)` | nil | углы + камера |
| `cmd:set_view_angles_silent(pitch, yaw, roll)` | nil | только usercmd |
| `cmd:get_forward_move()` | number | forward move |
| `cmd:set_forward_move(value)` | nil | установить forward move |
| `cmd:get_left_move()` | number | left move |
| `cmd:set_left_move(value)` | nil | установить left move |
| `cmd:get_up_move()` | number | up move |
| `cmd:set_up_move(value)` | nil | установить up move |
| `cmd:get_command_number()` | integer | command number |
| `cmd:get_tick()` | integer | tick |

`set_forward_move`, `set_left_move` и `set_up_move` помечают тик как
script-controlled: встроенное финальное subtick-обнуление movement
пропускается. Установка кнопок и углов такого флага не даёт.

## Глобальные таблицы

### z.api

```lua
z.api.version  -- "0.7.0"
z.api.game     -- "cs2"
z.api.script   -- абсолютный путь скрипта
z.api.ffi      -- true, если FFI включён
```

### z.globals

| Function | Return | Description |
|---|---|---|
| `z.globals.curtime()` | number или nil | игровое время |
| `z.globals.tick_count()` | integer или nil | номер тика |
| `z.globals.is_in_game()` | boolean | есть локальный pawn/observer |
| `z.globals.frame_time()` | number | delta кадра |
| `z.globals.fps()` | number | FPS xdraw |
| `z.globals.snapshot()` | table | все значения сразу |

`curtime` и `tick_count` возвращают `nil`, пока `global_vars` недоступны.

### z.time

```lua
z.time.now()    -- Unix time, секунды
z.time.tick()   -- секунды с запуска Windows
z.time.frame()  -- delta последнего кадра
```

### z.buttons

| Field | Value | Bit |
|---|---:|---:|
| `attack` | 1 | 0 |
| `jump` | 2 | 1 |
| `duck` | 4 | 2 |
| `forward` | 8 | 3 |
| `back` | 16 | 4 |
| `use` | 32 | 5 |
| `left` | 128 | 7 |
| `right` | 256 | 8 |
| `move_left` | 512 | 9 |
| `move_right` | 1024 | 10 |
| `second_attack` | 2048 | 11 |
| `reload` | 8192 | 13 |
| `sprint` | 65536 | 16 |
| `scoreboard` | 8589934592 | 33 |
| `zoom` | 17179869184 | 34 |
| `look_at_weapon` | 34359738368 | 35 |

### z.log

```lua
z.log("text")
z.log.info("text")
z.log.warn("text")
z.log.error("text")
print("text")
```

Аргументы приводятся к строке и соединяются пробелом. `print` — алиас
информационного лога. Лимита длины сообщения и числа записей нет.

## z.math

Вектор — `{ x, y, z }` плюс `[1]`, `[2]`, `[3]`. Угол — `{ pitch, yaw, roll }`
плюс индексы. Углы в градусах.

| Function | Return | Description |
|---|---|---|
| `z.math.vec3(x, y, z)` | vector | создать вектор |
| `z.math.angle(pitch, yaw[, roll])` | angle | создать угол |
| `z.math.angle_vectors(angle)` | forward, right, up | направления |
| `z.math.vector_angles(vector)` | angle | направление → углы |
| `z.math.calc_angle(src, dst)` | angle | угол из `src` в `dst` |
| `z.math.calc_fov(a, b)` | number | разница углов |
| `z.math.normalize_angle(angle)` | angle | нормализовать pitch/yaw, roll = 0 |
| `z.math.normalize_yaw(yaw)` | number | yaw в `-180..180` |
| `z.math.distance(a, b)` | number | расстояние |
| `z.math.length(vector)` | number | длина вектора |

## z.entitylist

Все функции возвращают копии. Pawn/controller-указатели, handle-ы и адреса
наружу не отдаются.

### Игроки

```lua
z.entitylist.players()        -- живые, по controller index
z.entitylist.local_player()   -- локальный/наблюдаемый pawn или nil
```

`players()` возвращает только живых игроков (`m_bPawnIsAlive` и `health > 0`),
отсортированных по `index`. Локальный отмечен `local = true`. `index` —
controller index, а не pawn и не слот scoreboard. В observer-режиме
`local_player()` описывает наблюдаемый pawn с `observer = true`.

Снимок игрока:

| Field | Type | Description |
|---|---|---|
| `index` | integer | controller index; только у `players()`/`entity()` |
| `local` | boolean | локальный игрок; только у `players()`/`entity()` |
| `alive` | boolean | жив |
| `observer` | boolean | observer-режим |
| `scoped` | boolean | в прицеле |
| `health` | integer | здоровье |
| `team` | integer | команда |
| `armor` | integer | броня |
| `ping` | integer | ping |
| `name` | string или nil | имя |
| `weapon` | string или nil | оружие без `weapon_` |
| `weapon_id` | integer | item definition index |
| `ammo` | integer | патроны |
| `max_ammo` | integer | размер магазина |
| `origin` | vector или nil | позиция pawn |
| `eye_position` | vector или nil | позиция глаз |
| `velocity` | vector или nil | `m_vecVelocity` |
| `abs_velocity` | vector или nil | `m_vecAbsVelocity` |
| `flags` | integer | `m_fFlags` |
| `on_ground` | boolean | `FL_ONGROUND` |
| `ducking` | boolean | `FL_DUCKING` |
| `sim_time` | number или nil | simulation time |

### Сущности

```lua
z.entitylist.all([include_players]) -- все не-игровые, по index
z.entitylist.entities([include_players])
z.entitylist.items()                -- type = item
z.entitylist.projectiles()          -- type = projectile
z.entitylist.others()               -- type = other
z.entitylist.entity(index)          -- player или общий снимок
z.entitylist.get(index)
```

`all()` исключает player controller и player pawn. `all(true)` добавляет живых
игроков в конец списка, по controller index. `items()`, `projectiles()` и
`others()` фильтруют по типу. `entity()` принимает индекс закэшированной
сущности.

Снимок сущности:

| Field | Type | Description |
|---|---|---|
| `index` | integer | индекс сущности |
| `type` | string | `item`, `projectile`, `other`, `player` |
| `type_id` | integer | числовой тип |
| `is_player` | boolean | player controller |
| `schema` | string или nil | schema name |
| `pawn_schema` | string или nil | schema pawn |
| `health` | integer | здоровье |
| `alive` | boolean | `health > 0` |
| `team` | integer | команда |
| `flags` | integer | `m_fFlags` |
| `on_ground` | boolean | `FL_ONGROUND` |
| `ducking` | boolean | `FL_DUCKING` |
| `dormant` | boolean или nil | `m_bDormant` |
| `origin` | vector или nil | позиция |
| `eye_position` | vector или nil | позиция глаз |
| `velocity` | vector или nil | скорость |
| `abs_velocity` | vector или nil | абсолютная скорость |
| `sim_time` | number или nil | simulation time |

### История движения

```lua
local samples = z.entitylist.history(index[, limit])
```

Возвращает motion-снимки от старых к новым. `limit` — `1..64`, по умолчанию
`16`. История собирается в `create_move` только для живых игроков. Поля:
`index`, `tick`, `time`, `health`, `team`, `origin`, `eye_position`, `velocity`,
`abs_velocity`, `flags`, `on_ground`, `ducking`, `sim_time`.

### Кости

```lua
z.entitylist.bones(index)             -- массив всех костей
z.entitylist.bone(index[, bone])      -- одна кость, по умолчанию head
z.entitylist.bone_count(index)        -- число костей или 0
```

`index` — controller index живого игрока, индекс живого player-pawn или индекс
не-игровой сущности. `bone` — id `0..65535` или имя:

```text
pelvis, spine_1..spine_4, neck, head,
left_clavicle, left_shoulder, left_elbow, left_hand,
right_clavicle, right_shoulder, right_elbow, right_hand,
left_hip, left_knee, left_foot, right_hip, right_knee, right_foot
```

Кость:

```lua
{
    id = 6,
    name = "head", -- может отсутствовать для неизвестного id
    position = { x = 0, y = 0, z = 0 },
    scale = 1.0,
    rotation = { x = 0, y = 0, z = 0, w = 1 },
}
```

### Хитбоксы

```lua
z.entitylist.hitboxes(index)          -- массив хитбоксов
z.entitylist.hitbox(index, hitbox_id) -- один хитбокс или nil
```

| Field | Type | Description |
|---|---|---|
| `index` | integer | индекс в наборе |
| `bone` | integer | bone id |
| `hitgroup` | integer | hitgroup |
| `hitgroup_name` | string | имя hitgroup |
| `mins` | vector | локальные mins |
| `maxs` | vector | локальные maxs |
| `radius` | number | радиус капсулы |
| `shape` | integer | shape type; только у массива |
| `translation_only` | boolean | флаг; только у массива |
| `origin` | vector или nil | центр в мире |

### Видимость

```lua
z.entitylist.visible(index[, bone])        -- boolean
z.entitylist.can_see(index[, bone])
z.entitylist.visible_hitbox(index, hitbox_id)
```

Проверяет точку кости (по умолчанию `head`) или центр хитбокса из глаза
локального/наблюдаемого pawn. Это не autowall и не проверка всего тела. Если
кость не найдена, используется `eye_position`, затем `origin`; при
недоступном trace возвращается `false`.

### Удобные функции

```lua
z.entitylist.get_local_player()
z.entitylist.get_local_player_pawn()
z.entitylist.get_local_player_name()
z.entitylist.get_local_weapon()
z.entitylist.get_local_weapon_name()
z.entitylist.get_local_weapon_index()
```

`get_local_weapon()` возвращает `{ name, id, ammo, max_ammo }` или `nil`.

## z.combat

### z.combat.autowall(start, end[, target])

Без `target` — проверка пробития в направлении `end - start` через
`g_shared.pen().can()`. С `target` — самая новая валидная lagcomp-запись,
`prepare_target()` и `pen().run()`. `target` — controller index или индекс
живого player-pawn.

```lua
local result = z.combat.autowall(start_pos, end_pos, player.index)
```

Без цели:

| Field | Type | Description |
|---|---|---|
| `hit` | boolean | урон больше нуля |
| `damage` | number | оценка урона |
| `penetrated` | boolean | совпадает с `hit` |
| `target` | integer | `0` |

С целью:

| Field | Type | Description |
|---|---|---|
| `hit` | boolean | попадание |
| `damage` | number | урон |
| `penetrated` | boolean | пробита ли поверхность |
| `target` | integer | переданный индекс |
| `hitbox` | integer | индекс или `-1` |
| `hitgroup` | integer | hitgroup или `-1` |
| `hitgroup_name` | string | имя или `"none"` |
| `tick` | integer | tick записи |

При мёртвом локальном игроке, не-игровой цели или отсутствии записей —
`hit = false`, `damage = 0` и `reason`. `start` и `end` должны различаться.

### z.combat.backtrack(index)

Возвращает самую новую валидную запись или `nil`. Поля: `index`, `tick`,
`sim_time`, `valid`, `was_valid`, `extrapolated`, `flags`, `origin`,
`bone_count`, `bones`. Каждая кость — `{ id, name?, position, scale, rotation }`.
Костей в записи не больше `128`.

### z.combat.backtrack_records(index[, limit])

Возвращает валидные записи от старых к новым. `limit` по умолчанию `16`,
минимум `1`; если записей меньше — все. Формат тот же, включая `index`.

Алиасы: `z.combat.latest_record()` = `backtrack()`,
`z.combat.can_shoot()` = `autowall()`.

## z.trace

```lua
z.trace.line(start, end[, options])
z.trace.hull(start, end, mins, maxs[, options])
```

| Option | Type | Default | Description |
|---|---|---|---|
| `skip_local` | boolean | `true` | не трассировать локального pawn |
| `mask` | integer | `0x1c3003` | contents mask |
| `layer` | integer | `4` | collision layer |

Результат:

| Field | Type | Description |
|---|---|---|
| `hit` | boolean | попадание |
| `fraction` | number | доля отрезка `0..1` |
| `position` | vector | точка попадания |
| `normal` | vector | нормаль |
| `contents` | integer | contents |
| `all_solid` | boolean | старт в solid |
| `hit_player` | boolean | попали в pawn |
| `hit_player_index` | integer или nil | controller index |

Один вызов не пробивает стены. Для penetration используй `z.combat.autowall`.

## z.engine

| Function | Return | Description |
|---|---|---|
| `z.engine.is_connected()` | boolean | engine client и `global_vars` готовы |
| `z.engine.is_in_game()` | boolean | есть локальный pawn/observer |
| `z.engine.client_cmd(command)` | nil | выполнить client command |
| `z.engine.prestate()` | table | последнее prediction-состояние |
| `z.engine.predict_state([cmd])` | table | выполнить prediction и вернуть снимок |
| `z.engine.predict(fn[, cmd])` | true | выполнить callback в prediction |

`client_cmd` выполняет команду сразу. Это не movement API: доступны `quit`,
`disconnect`, `bind` и любые другие клиентские команды.

`predict_state` — синхронный вариант: запускает `g_prediction.simulate()` и
возвращает снимок предсказанного локального pawn. Поля: `predicted`, `health`,
`team`, `flags`, `on_ground`, `ducking`, `origin`, `eye_position`, `velocity`,
`abs_velocity`, `sim_time`. Без `cmd` работает только в `create_move`.

`predict` без `cmd` работает только внутри `create_move`. Вне его передай
временный `cmd` явно. Вложенный prediction запрещён. Callback получает
предсказанное состояние через `z.entitylist.local_player()` и `prestate`.
Алиас: `z.engine.simulate()`.

`prestate()` всегда возвращает таблицу: `flags`, `on_ground`, `ducking`,
`networked_velocity`, `velocity`, `origin`, `networked_origin`,
`last_movement_impulses`, `surface_friction`, `stamina`. До первого
`simulate()` значения нулевые.

## z.render

`z.render` работает в `paint` и вне его. Внутри `paint` команды идут в текущий
кадр. Вне `paint` они складываются в отложенную очередь и отправляются в начале
следующего `on_present`, то есть появляются на кадр позже. До создания
D3D11-устройства выбрасывается
`render is unavailable before the game renderer is initialized`.

Цвет — `{ r, g, b, a }` (недостающие поля `255`, значения обрезаются до
`0..255`) или упакованное `0xAARRGGBB`.

| Function | Description |
|---|---|
| `z.render.rect_filled(x, y, w, h, color[, radius])` | заполненный прямоугольник |
| `z.render.rect_filled_gradient(x, y, w, h, tl, tr, br, bl[, radius])` | градиент по углам |
| `z.render.rect_filled_blurred(x, y, w, h[, tint[, radius]])` | blur фона |
| `z.render.rect(x, y, w, h, color[, thickness[, radius]])` | рамка |
| `z.render.line(x0, y0, x1, y1, color[, thickness])` | линия |
| `z.render.circle(x, y, radius, color[, thickness])` | окружность |
| `z.render.circle_filled(x, y, radius, color)` | круг |
| `z.render.triangle_filled(x0, y0, x1, y1, x2, y2, color)` | треугольник |
| `z.render.polyline(points, color[, closed[, thickness]])` | ломаная |
| `z.render.polyline_gradient(points, colors[, closed[, thickness]])` | цвета по точкам |
| `z.render.convex_filled(points, color)` | выпуклый многоугольник |
| `z.render.text(x, y, text, color[, font][, style])` | текст |
| `z.render.measure_text(text[, font])` | `width, height` |
| `z.render.screen_size()` | `width, height` |
| `z.render.world_to_screen(x, y, z)` | `x, y` или `nil` |
| `z.render.world_to_screen(vector)` | `x, y` или `nil` |
| `z.render.image(x, y, w, h, image[, tint[, radius]])` | изображение |
| `z.render.image_uv(x, y, w, h, image, u0, v0, u1, v1[, tint])` | изображение с UV |

`style` — `"normal"`, `"outlined"`, `"shadowed"`. `points` — массив
`{ x, y }` или плоский `{ x1, y1, x2, y2, ... }`; для polyline нужно минимум
две точки, для convex — три.

### Градиент

Порядок цветов: **top-left, top-right, bottom-right, bottom-left**.
`polyline_gradient` берёт цвет на каждую точку; если цветов меньше, последний
повторяется.

### Clip и шрифты

```lua
z.render.push_clip(x, y, w, h)          -- пересечь с текущим clip
z.render.push_clip_absolute(x, y, w, h) -- заменить текущий clip
z.render.pop_clip()
z.render.push_font(font)
z.render.pop_font()
```

Clip — состояние draw-list. Всегда закрывай его. `push_font` меняет глобальный
стек xdraw, а не команду списка; вне `paint` обязательно вызывай `pop_font` в
том же обработчике.

### Draw-list

```lua
local list = z.render.new_list()
list:rect_filled(20, 20, 200, 60, { r = 12, g = 12, b = 18, a = 220 }, 8)
list:text(32, 38, "cached", { r = 255, g = 255, b = 255, a = 255 })

z.events.paint:add(function()
    z.render.submit(list) -- или list:submit()
end)
```

`z.render.create_list()` — алиас `new_list`. Методы списка повторяют
`z.render` для примитивов, текста, изображений и clip. Дополнительно:

| Method | Description |
|---|---|
| `list:clear()` | очистить вершины и команды |
| `list:append(other)` | добавить копию другого списка |
| `list:submit()` / `list:draw()` | отправить в кадр или очередь |
| `tostring(list)` | `draw_list(N vertices, M commands)` |

У списка нет `measure_text`, `screen_size`, `world_to_screen`, `push_font` и
`pop_font`. `submit` не очищает список. Список можно отправлять несколько раз.

### Отложенный рендер

Команды вне `paint` появляются в начале следующего `on_present` до Lua `paint`.
Если игра не представляет кадры, очередь растёт. Clip входит в данные
draw-list; незакрытый clip обрежет последующие команды.

## z.assets

```lua
local image, err = z.assets.load_image(path[, scale])
local image, err = z.assets.load_svg(svg_text[, scale])
local font, err = z.assets.load_font(path, size[, atlas_w, atlas_h])
```

Алиасы: `z.assets.image()` = `load_image()`, `z.assets.svg()` = `load_svg()`,
`z.assets.font()` = `load_font()`. В `load_image` параметр `scale` учитывается
только для `.svg`; для растровых форматов он игнорируется. Относительный путь
проверяется от папки скриптов, затем от рабочей папки; абсолютный принимается
как есть. Ассеты могут читаться вне папки скриптов. Загрузка синхронная.

`image`:

| Method | Return | Description |
|---|---|---|
| `image:valid()` | boolean | текстура доступна |
| `image:size()` | width, height | размеры |
| `image:frames()` | integer | число кадров |

`font`:

| Method | Return | Description |
|---|---|---|
| `font:measure(text)` | width, height | размеры строки |
| `font:size()` | size, line_height | размер и высота строки |

При ошибке — `nil, error`. Изображения удерживаются keepalive; шрифты
принадлежат xdraw. После reload загрузи токены заново.

## z.net

Все вызовы синхронные и блокируют поток. Сеть работает и без FFI.

### HTTP

```lua
local response, err = z.net.http(url_or_options)
```

| Option | Type | Default | Description |
|---|---|---|---|
| `url` | string | — | URL |
| `method` | string | `"GET"` | HTTP-метод |
| `body` | string | `""` | тело |
| `headers` | table | `{}` | заголовки |
| `timeout` | integer | `10000` | мс, `1..600000` |

Ответ: `status`, `ok`, `body`, `url`, `raw_headers`, `headers` (ключи в нижнем
регистре). HTTPS проверяет сертификат через WinHTTP, используется системный
proxy и автоматические редиректы. Алиас: `z.net.request()`.

### DNS

```lua
local addresses, err = z.net.dns(host)
```

Массив числовых IPv4/IPv6 адресов. Алиас: `z.net.resolve()`.

### TCP

```lua
local socket, err = z.net.tcp_connect(host, port[, timeout])
socket:send(data)
local data = socket:recv(max_bytes)
socket:set_timeout(ms)
socket:close()
```

`timeout` — `1..600000` мс, по умолчанию `5000`. После подключения таймаут
применяется к `send`/`recv`. Алиас: `z.net.connect()`.

### UDP

```lua
local socket, err = z.net.udp([timeout])
socket:send_to(data, host, port)
local data, host, port = socket:recv_from(max_bytes)
```

Сначала пробуется IPv6 с `IPV6_V6ONLY=0`, затем IPv4. `recv_from` возвращает
числовой адрес и integer port.

Один `recv`/`recv_from` — не больше `64 MiB`. После `close()` методы сокета
выбрасывают `socket is closed`. TCP не использует TLS; для HTTPS нужен
`z.net.http`. Сокеты закрываются GC, но лучше закрывать вручную.

## z.settings

Доступ к реестру настроек по `category, name`. Поддерживаются обе формы:

```lua
z.settings.get("ragebot - rifle", "hit chance")
z.settings.get("ragebot - rifle.hit chance")
```

Короткая форма делится по первой точке. `category`/`name` — `1..120` байт,
путь — `3..241` байт.

```lua
z.settings.list([category]) -- { { category, name, type }, ... }
z.settings.exists(category, name)
z.settings.type(category, name)
z.settings.get(category, name)
z.settings.set(category, name, value) -- true
```

Типы: `bool`, `int`, `float`, `color` (`{ r, g, b, a }`), `vec3`
(`{ x, y, z }`), `bool[]`, `string`, `custom`. Для `uint8` значения обрезаются
до `0..255`; `float`/`vec3` требуют конечных чисел; строки не имеют лимита
`4096`.

### Custom-поля

Custom-поля читаются как таблица/массив/скаляр и записываются целиком, без
слияния. Таблица `1..n` без дыр — JSON-массив, остальные — объект. Ключи —
строки или числа, `nil` → `null`, циклы запрещены.

```lua
-- changer.applied skins:
-- { ["7"] = { p = 524, w = 0.01, s = 1, t = 0, d = 0 } }

local agents = z.settings.get("changer", "agents")
agents.ct = 0
agents.t = 0
z.settings.set("changer", "agents", agents)

-- changer.custom agents:
-- { selected_ct = 0, selected_t = 0,
--   entries = { { name = "...", model_path = "...", team = 3 } } }

local list = z.settings.get("lua", "script list")
list[#list + 1] = { path = "new.lua", enabled = true }
z.settings.set("lua", "script list", list) -- reload будет запрошен
```

`changer.applied skins` — объект, ключ — definition index оружия строкой,
значение — `{ p, w, s, t, d }`. `changer.agents` — `{ ct, t }`.
`changer.custom agents` — `{ selected_ct, selected_t, entries }`, `team`:
`2 = T`, `3 = CT`.

## z.scripts

```lua
z.scripts.reload()
z.scripts.directory()
z.scripts.open_directory()
```

`reload()` ставит перезагрузку в очередь; она выполняется вне активного
callback. У каждого скрипта вызывается `shutdown`, кроме упавших. Текущий
скрипт может выключить себя через `lua.script list`; код после `set` успеет
выполниться.

`directory()` возвращает абсолютный путь папки скриптов.
`open_directory()` открывает её в Explorer и ничего не возвращает.

## z.ui

`z.ui` работает только внутри `z.events.ui`. Окна используют текущий `xui`,
тему и clipping; это не второй ImGui context.

```lua
z.ui.begin_window(title, x, y, w, h[, resizable])
z.ui.end_window()
z.ui.set_window_rect(title, x, y, w, h)
z.ui.text(text[, color])
z.ui.separator()
z.ui.same_line([offset])
z.ui.spacing([amount])
z.ui.button(label[, width, height])   -- boolean
z.ui.checkbox(label, value)           -- boolean
z.ui.slider(label, value, min, max[, format])     -- number
z.ui.slider_int(label, value, min, max[, format]) -- integer
z.ui.combo(label, value, items[, width])          -- integer
z.ui.text_input(label, value[, max_len[, hint]])  -- string
z.ui.screen_size()                    -- width, height
z.ui.measure_text(text)               -- width, height
```

Позиция и размер окна сохраняются в state. Окно нужно закрыть до возврата из
обработчика; незакрытое закроется принудительно с предупреждением. Заголовок
уникален в пределах скрипта. Минимальный размер окна — `160×100`.

## FFI

При `allow ffi (unsafe)` открываются стандартные библиотеки LuaJIT, включая
`ffi`, `io`, `os`, `debug` и `package`. `package.path` начинается с папки
скриптов. При выключенном FFI нет `ffi`, `io`, `os`, `debug`, `package`,
`dofile`, `loadfile`; `require` всегда выбрасывает ошибку.

```lua
local ffi = require("ffi")
ffi.cdef[[
    int MessageBoxA(void* hwnd, const char* text, const char* caption, unsigned type);
]]
ffi.C.MessageBoxA(nil, "hello", "z", 0)
```

FFI даёт полный доступ к памяти процесса. Не запускай чужие FFI-скрипты.

## Ограничения

Искусственные пределы сняты:

| Parameter | Status |
|---|---|
| Размер `.lua` | нет лимита |
| Записей в списке скриптов | нет лимита |
| Длина пути скрипта | ограничена файловой системой |
| Лог в памяти | без фиксированного числа |
| Лог в UI | все текущие записи |
| Длина сообщения | без лимита |
| Длина client command | без лимита, кроме памяти и NUL |
| Строковые настройки | без лимита `4096` |

Практические границы: history `64` (по умолчанию `16`), backtrack по умолчанию
`16`, `recv` `64 MiB`, network timeout `1..600000` мс, окно минимум `160×100`,
settings path `1..120`/`3..241` байт. `backtrack_records` не имеет верхнего
искусственного предела.

Скрипт всё равно выполняется в процессе игры: бесконечный цикл заблокирует
поток, а лог/очередь рендера могут занять память.

## Примеры

### Bunny hop

```lua
z.events.create_move:add(function(cmd)
    cmd:set_button(z.buttons.jump, true)
end)
```

### Overlay

```lua
z.events.paint:add(function()
    z.render.rect_filled(20, 20, 190, 42, { r = 18, g = 18, b = 22, a = 220 }, 6)
    z.render.text(32, 32, "zantikira lua", { r = 255, g = 255, b = 255, a = 255 })
end)
```

### Локальный игрок

```lua
z.events.paint:add(function()
    local p = z.entitylist.local_player()
    if not p or not p.alive then
        return
    end

    z.render.text(20, 20, ("%s | %d hp | %s"):format(
        p.name or "unknown", p.health, p.weapon or "none"
    ), { r = 255, g = 255, b = 255, a = 255 })
end)
```

### ESP по костям

```lua
z.events.paint:add(function()
    for _, player in ipairs(z.entitylist.players()) do
        if not player.local and player.origin then
            local head = z.entitylist.bone(player.index, "head")
            if head then
                local x, y = z.render.world_to_screen(head.position)
                if x and y then
                    z.render.text(x, y, player.name or "unknown", {
                        r = 255, g = 80, b = 80, a = 255
                    })
                end
            end
        end
    end
end)
```

### Autowall по backtrack

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
                local result = z.combat.autowall(
                    me.eye_position, record.origin, player.index)
                if result.hit and result.damage > 0 then
                    z.log(player.name, result.damage)
                end
            end
        end
    end
end)
```

### Prediction

```lua
z.events.create_move:add(function()
    z.engine.predict(function()
        local me = z.entitylist.local_player()
        if me and me.origin then
            z.log(("predicted z = %.2f"):format(me.origin.z))
        end
    end)
end)
```

### HTTP

```lua
local response, err = z.net.http("https://example.com/", { timeout = 3000 })
if response and response.ok then
    z.log(response.status, #response.body)
else
    z.log.error(err)
end
```

### Draw-list

```lua
local list = z.render.new_list()
list:rect_filled(20, 20, 240, 80, { r = 12, g = 12, b = 18, a = 220 }, 8)
list:text(32, 38, "cached", { r = 255, g = 255, b = 255, a = 255 })

z.events.paint:add(function()
    z.render.submit(list)
end)
```

### Своё окно

```lua
z.events.ui:add(function()
    z.ui.begin_window("my panel", 40, 40, 320, 220, true)
    z.ui.text("zantikira lua")
    if z.ui.button("reload", 120, 24) then
        z.scripts.reload()
    end
    z.ui.end_window()
end)
```
