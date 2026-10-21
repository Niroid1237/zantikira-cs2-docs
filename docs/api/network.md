# network

`z.net` — синхронный HTTP-клиент и сырые TCP/UDP-сокеты. Все вызовы блокируют
поток, из которого запущен Lua, до ответа, таймаута или ошибки. Не вызывай их
каждый кадр в `paint`/`create_move` без необходимости.

Сеть работает и при выключенном FFI. Это осознанное расширение API: скрипт
может отправлять данные в интернет и принимать ответы. Запускай только свой код.

## z.net.http(url[, options])

```lua
local response, err = z.net.http("https://example.com/api", {
    method = "GET",
    timeout = 10000,
    headers = {
        ["Accept"] = "application/json",
        ["Authorization"] = "Bearer token",
    },
})
```

Можно передать просто URL строкой:

```lua
local response, err = z.net.http("https://example.com/")
```

Алиас: `z.net.request()`.

| Option | Type | Default | Description |
|---|---|---|---|
| `url` | string | — | URL, обязателен |
| `method` | string | `"GET"` | HTTP-метод |
| `body` | string | `""` | тело запроса |
| `headers` | table | `{}` | дополнительные заголовки |
| `timeout` | integer | `10000` | миллисекунды, `1..600000` |

URL обязан содержать схему и хост. HTTPS проверяет сертификат системным
хранилищем WinHTTP. Используется системный proxy и автоматические редиректы.

### Ответ

Успешный ответ — таблица:

| Field | Type | Description |
|---|---|---|
| `status` | integer | HTTP status code |
| `ok` | boolean | `status` в диапазоне `200..299` |
| `body` | string | тело ответа |
| `url` | string | исходный URL |
| `raw_headers` | string | сырые заголовки с `\r\n` |
| `headers` | table | заголовки с ключами в нижнем регистре |

При ошибке транспорта/URL возвращается `nil, error`.

```lua
local response, err = z.net.http("https://example.com/")
if not response then
    z.log.error(err)
elseif response.ok then
    z.log(response.status, #response.body)
end
```

Тело не ограничено искусственным лимитом. Огромный ответ занимает память
процесса.

## z.net.dns(host)

```lua
local addresses, err = z.net.dns("example.com")
if addresses then
    for _, ip in ipairs(addresses) do
        z.log(ip)
    end
end
```

Возвращает массив числовых адресов (`IPv4`/`IPv6`) или `nil, error`. Алиас:
`z.net.resolve()`. Вызов использует системный resolver и может блокироваться.

## z.net.tcp_connect(host, port[, timeout])

```lua
local socket, err = z.net.tcp_connect("example.com", 80, 5000)
if not socket then
    z.log.error(err)
    return
end

socket:send("GET / HTTP/1.0\r\nHost: example.com\r\n\r\n")
local data = socket:recv(65536)
socket:close()
```

Возвращает сокет или `nil, error`. `timeout` — миллисекунды, `1..600000`, по
умолчанию `5000`. Соединение устанавливается синхронно; после подключения
таймаут применяется к `send`/`recv`.

Алиас: `z.net.connect()`.

### socket

| Method | Return | Description |
|---|---|---|
| `socket:send(data)` | integer или nil, error | отправить все байты |
| `socket:recv(max_bytes?)` | string или nil, error | принять до `max_bytes`, по умолчанию `65536` |
| `socket:set_timeout(ms)` | — | изменить receive/send timeout |
| `socket:close()` | — | закрыть сокет |
| `tostring(socket)` | string | `socket(open)` или `socket(closed)` |

`recv` возвращает `nil, "closed"`, если удалённая сторона закрыла соединение.
При таймауте возвращается `nil, "timeout"` или системный текст ошибки.

Максимальный размер одного `recv`/`recv_from` — `64 MiB`. Это защита от
случайного выделения памяти, а не ограничение скрипта.

## z.net.udp([timeout])

```lua
local socket, err = z.net.udp(5000)
if not socket then
    z.log.error(err)
    return
end

socket:send_to("ping", "127.0.0.1", 27015)
local data, host, port = socket:recv_from()
z.log(host, port, data)
socket:close()
```

Создаёт UDP-сокет. Сначала пытается `IPv6` с `IPV6_V6ONLY=0`, затем `IPv4`.
`timeout` — миллисекунды, по умолчанию `5000`.

| Method | Return | Description |
|---|---|---|
| `socket:send_to(data, host, port)` | integer или nil, error | отправить датаграмму |
| `socket:recv_from(max_bytes?)` | data, host, port или nil, error | принять датаграмму |

`host` в `recv_from` — числовой адрес, `port` — integer.

## Таймауты и ошибки

- Все таймауты ограничены `1..600000` мс (`10 минут`).
- Таймаут — не гарантия мгновенного возврата: DNS и установка соединения могут
  занять дополнительное время.
- TCP не использует TLS. Для HTTPS нужен `z.net.http`.
- Сокеты закрываются сборщиком мусора, но лучше закрывать вручную.
- После `close()` любой метод сокета выбрасывает ошибку `socket is closed`.
- Сеть не является песочницей. Не передавай токены и не выполняй команды из
  ответа без проверки.

## Пример: JSON API

```lua
local response, err = z.net.http({
    url = "https://api.example.com/status",
    method = "GET",
    timeout = 3000,
})

if response and response.ok then
    -- JSON-парсера в API нет; body остаётся строкой.
    z.log(response.body)
else
    z.log.error(err or ("HTTP " .. tostring(response.status)))
end
```

JSON-парсер и clipboard в API не входят. При включённом FFI можно подключить
свою библиотеку, но это уже ответственность скрипта.
