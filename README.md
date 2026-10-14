# Лабораторная работа #3

![GitHub Classroom Workflow](../../workflows/GitHub%20Classroom%20Workflow/badge.svg?branch=master)

## Fault Tolerance

### Формулировка

На базе [Лабораторной работы #2](https://github.com/bmstu-rsoi/lab2-template) реализовать механизмы, увеличивающие
отказоустойчивость системы.

### Требования

1. На Gateway Service для _всех операций_ чтения реализовать паттерн Circuit Breaker. Накапливать статистику в памяти, и
   если система не ответила N раз, то в N + 1 раз вместо запроса сразу отдавать fallback. Через небольшой timeout
   выполнить запрос к реальной системе, чтобы проверить ее состояние.
2. На каждом сервисе сделать специальный endpoint `GET /manage/health`, отдающий 200 ОК, он будет использоваться для
   проверки доступности сервиса (в [Github Actions](.github/workflows/classroom.yml) в скрипте проверки готовности всех
   сервисов [wait-script.sh](scripts/wait-script.sh) и в тестах [test-script.sh](scripts/test-script.sh)).
   ```shell
   "$path"/wait-for.sh -t 120 "http://localhost:$port/manage/health" -- echo "Host localhost:$port is active"
   ```
4. В случае недоступности данных из некритичного источника (не основного), возвращается fallback-ответ. В зависимости от
   ситуации, это может быть:
    * пустой объект или массив;
    * объект, с заполненным полем (`uid` или подобным), по которому идет связь с другой системой;
    * default строка (если при этом не меняется тип переменной).
5. В задании описаны две операции, изменяющие состояния нескольких систем. В случае недоступности одной из систем,
   участвующих в этой операции, выполнить:
    1. откат всей операции;
    2. возвращать пользователю ответ об успешном завершении операции, а на Gateway Service поставить этот запрос в
       очередь для повторного выполнения.
6. Для автоматических прогонов тестов в файле [autograding.json](.github/classroom/autograding.json)
   и [classroom.yml](.github/workflows/classroom.yml) заменить `<variant>` на ваш вариант.
7. В [docker-compose.yml](docker-compose.yml) прописать сборку и запуск docker контейнеров.
8. Код хранить на Github, для сборки использовать Github Actions.
9. Каждый сервис должен быть завернут в docker.
10. В classroom.yml дописать шаги на сборку, прогон unit-тестов.

### Пояснения

1. Для локальной разработки можно использовать Postgres в docker.
2. Схема взаимодействия сервисов остается как в [Лабораторной работы #2](https://github.com/bmstu-rsoi/lab2-template).
3. Для реализации очереди можно использовать language native реализацию (например, BlockingQueue для Java), либо
   какую-то готовую реализацию типа RabbitMQ, Redis, ZeroMQ и т.п. Крайне нежелательно использовать реляционную базу
   данных как средство эмуляции очереди.
4. Можно использовать внешнюю очередь или запускать ее в docker.
5. Для проверки отказоустойчивости используется остановка и запуск контейнеров docker, это делает
   скрипт [test-script.sh](scripts/test-script.sh). Скрипт нужно запускать из корня проекта, т.к. он обращается к папке
   postman по вариантам.
   ```shell
   # запуск тестового сценария:
   # * <variant> – номер варианта (v1 | v2 | v3 | v4 )
   # * <service> – имя сервиса в Docker Compose
   # * <port>    – порт, на котором запущен сервис
   $ scripts/test-script.sh <variant> <service> <port>
   ```

### Прием задания

1. При получении задания у вас создается fork этого репозитория для вашего пользователя.
2. После того как все тесты успешно завершатся, в Github Classroom на Dashboard будет отмечено успешное выполнение
   тестов.

### Варианты заданий

Распределение вариантов заданий аналогично [ЛР #2](https://github.com/bmstu-rsoi/lab2-template).

1. [Flight Booking System](v1/README.md)
2. [Hotels Booking System](v2/README.md)
3. [Car Rental System](v3/README.md)
4. [Library System](v4/README.md)

## Gateway read Circuit Breaker

All outgoing GET and HEAD requests share an in-memory Circuit Breaker transport,
with independent state for each downstream service URL. This includes both
reverse-proxy endpoints and the reads used to assemble reservations and user data.

- Closed: requests reach the service; three consecutive network failures or HTTP
  5xx responses open its circuit. HTTP 4xx responses do not count as outages.
- Open: subsequent reads fail immediately for 10 seconds, using the existing
  Gateway unavailable response. No request reaches that service.
- Half-open: after the timeout, one read probes the service. Success closes the
  circuit; failure starts another 10-second wait. Concurrent probes are rejected.

State is protected by mutexes. Older in-flight responses cannot close a newly
opened circuit. Writes bypass the read breaker. State resets when Gateway restarts.
The constants are in `gateway-service/circuit_breaker.go`.

Manual check (from the repository root):

```powershell
docker compose up -d --build gateway-service
docker compose stop loyalty-service
1..4 | ForEach-Object { curl.exe -i -H "X-User-Name: Test Max" http://localhost:8080/api/v1/loyalty }
docker compose logs --tail 30 gateway-service
docker compose start loyalty-service
```

Wait for Loyalty to become healthy and for the breaker timeout to expire, then
repeat the GET. Expect 200 once the service has recovered. Gateway logs show
when the circuit opens and when a successful probe closes it.

Retrying canceled-booking loyalty
updates is the next implementation step. The complete instructor failover
scenario should be run after those steps are implemented.

## Optional data fallback

GET /api/v1/reservations and GET /api/v1/reservations/{reservationUid} return
HTTP 200 with payment: {} when the Payment read fails or its circuit is open.
GET /api/v1/me also returns loyalty: {} when the Loyalty read fails. Successfully
fetched reservation and hotel fields are retained. Failed decodes discard partial
optional values. Valid zero prices, discounts and counters remain in normal responses.

Reservation and hotel reads remain required. A direct GET /api/v1/loyalty still
returns an error when Loyalty is unavailable: there is no primary data to return.
This fallback applies to reads; it does not turn failed bookings or cancellations
into successful operations.

After rebuilding Gateway, create a booking, stop payment-service, and GET the
booking again. Expect 200 and payment: {}. Start payment-service afterward and
allow the circuit timeout to expire. Likewise stop loyalty-service and GET
/api/v1/me; reservations should remain available with loyalty: {}.
