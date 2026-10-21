# Модули API

Статус модулей в `0.7.0`. Имена `z.*` — публичный API ZantiKira; внутренняя
реализация может меняться без сохранения ABI.

| Module | ZantiKira | Status |
|---|---|---|
| events | `z.events` | готово |
| user command | `cmd` | готово |
| math | `z.math` | готово |
| render | `z.render` | готово: примитивы, градиенты, blur, clip, изображения, шрифты, draw-list, deferred |
| globals | `z.globals` | готово |
| time | `z.time` | готово |
| input buttons | `z.buttons` | готово |
| logging | `z.log`, `print` | готово, без искусственных лимитов |
| engine | `z.engine` | готово: состояние, `client_cmd`, prediction |
| entity list | `z.entitylist` | готово: игроки, все сущности, кости, хитбоксы, видимость, motion-история |
| combat | `z.combat` | готово: autowall, backtrack-записи |
| trace | `z.trace` | готово: луч и hull |
| menu / windows | `z.ui` | готово: базовые виджеты |
| settings | `z.settings` | готово: primitive и custom-поля |
| scripts | `z.scripts` | готово: reload и папка |
| assets | `z.assets` | готово: PNG/GIF/SVG, шрифты |
| network | `z.net` | готово: HTTP, DNS, TCP, UDP |
| cvars / netvars | — | не реализовано |
| JSON / clipboard | — | не реализовано |
| async / yield | — | не реализовано |

## Что осталось за пределами API

- `cvars`/`netvars`: произвольное чтение и запись консольных переменных не
  входит в публичный API. Для client commands есть `z.engine.client_cmd`, но
  это не типизированный cvar-интерфейс.
- JSON-парсер и clipboard: HTTP возвращает тело строкой; разбор и буфер обмена
  скрипт делает сам, при необходимости через FFI.
- Деревья, таблицы, color picker и keybind-виджеты в `z.ui` пока не появились.
- Асинхронные события и yield: все события синхронные, сеть блокирующая.
- Индивидуальные разрешения на скрипт: есть только глобальный FFI-флаг.
- Сырые указатели на сущности, pawn, текстуры и сокеты наружу не отдаются и не
  планируются.

## Ориентиры

- `z.entitylist` даёт снимки и копии костей; для собственной экстраполяции
  используй `history`, для lag compensation — `z.combat`.
- `z.combat` использует текущее оружие и `g_shared`; это не отдельный
  симулятор урона.
- `z.render` — слой xdraw поверх сцены и под меню; deferred-очередь показывает
  команды, вызванные вне `paint`, на следующем кадре.
- `z.ui` использует существующий `xui`, а не второй Dear ImGui context.
- `z.settings` работает через реестр конфига без raw-указателей.
- `z.net` и `z.assets` доступны без FFI и расширяют поверхность атаки; это
  осознанное решение, а не песочница.
