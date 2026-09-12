# Лабораторная работа №1. Метрики и Prometheus

> Этот файл — задание + согласованный план реализации. Контекст предыдущей сессии
> с Claude не сохранился, поэтому всё нужное для продолжения работы собрано здесь.

## Текущее состояние проекта (12.09.2026)

- Сгенерирован пустой проект через start.spring.io: Kotlin 2.3.21, **Spring Boot 4.1.1**, Gradle Kotlin DSL, Java 21.
- Группа `com.shestikpetr`, пакет `com.shestikpetr.ordersobservability`.
- Пока один модуль (`src/main/kotlin/.../OrdersObservabilityApplication.kt`), кода нет.
- Подключены стартеры (Boot 4 именование): `actuator`, `amqp`, `data-jdbc`, `flyway`, `validation`, `webmvc`,
  `flyway-database-postgresql`, `kotlin-reflect`, `tools.jackson.module:jackson-module-kotlin`,
  `micrometer-registry-prometheus`, `postgresql`.
- Git-репозиторий инициализирован.

## Что нужно сделать (план)

1. **Разбить на три Gradle-модуля**: `api`, `worker`, `exporter` (общий root `build.gradle.kts` с `apply false`,
   `settings.gradle.kts` с `include`). Пакеты: `com.shestikpetr.orders.api`, `...worker`, `...exporter`.
   Существующий `src/` из корня удалить/перенести в `api`.
2. **api** (порт 8080): `POST /orders`, `GET /orders/{id}`, `GET /orders`, `GET /health`, `GET /metrics`, `GET /internal/stats`.
   Создаёт заказ со статусом NEW, публикует `order_id` в RabbitMQ.
3. **worker** (порт 9101): `@RabbitListener`, NEW → PROCESSING → `Thread.sleep(PROCESSING_DELAY_MS)` → DONE + `processed_at`.
   Отдаёт `GET /metrics`.
4. **exporter** (порт 9102): по `@Scheduled` (раз в 5–10 с) дёргает `GET http://api:8080/internal/stats` через `RestClient`,
   выставляет 4 Gauge. Отдаёт `GET /metrics`.
5. **Docker**: три Dockerfile (multi-stage: `gradle:8-jdk21` → `eclipse-temurin:21-jre`), `docker-compose.yml` со всем стендом.
6. **Prometheus**: `prometheus/prometheus.yml` с 7 целями + `remote_write` в VictoriaMetrics.
7. **README.md**: списки метрик, 5–7 PromQL с пояснениями, вывод; место под скриншот Targets.

### Технические решения

| Вопрос | Решение |
|---|---|
| БД-доступ | Spring Data JDBC (data class `Order` с `@Table("orders")`), без JPA-плагинов |
| Схема | Flyway `V1__orders.sql`: `orders(id bigserial pk, amount numeric(12,2), items_count int, status text, created_at timestamptz, processed_at timestamptz null)` |
| Метрики HTTP API | Свой `OncePerRequestFilter`: `Counter("http_requests")` → `http_requests_total{method,route,status}` и `Timer("http_request_duration").publishPercentileHistogram()` → `http_request_duration_seconds_bucket/_sum/_count`. Route из `HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE` (`/orders/{id}`, не `/orders/42`). Встроенные `http_server_requests` Micrometer отключить, чтобы не дублировать |
| `shop_orders_created_total` | `Counter("shop_orders_created")`, инкремент после успешного INSERT |
| Метрика Worker | `Timer("shop_order_processing_duration").publishPercentileHistogram()` — от получения сообщения до UPDATE в DONE |
| `/metrics` и `/health` | Свои контроллеры: `/metrics` возвращает `PrometheusMeterRegistry.scrape()` с `Content-Type: text/plain; version=0.0.4`; `/health` — `{"status":"UP"}`. Actuator не переназначаем |
| `/internal/stats` | Один SQL: `count(*) filter (where status='NEW')`, аналогично PROCESSING/DONE, `max(processed_at)`. Через `/metrics` API статусы НЕ отдавать |
| Exporter gauges | `shop_stats_orders_new`, `shop_stats_orders_processing`, `shop_stats_orders_done`, `shop_stats_seconds_since_last_processed` (= now − last_processed_at, если null → NaN или не выставлять) |
| RabbitMQ | Очередь `orders.process`, сообщение — JSON `{"order_id": 42}`. Образ `rabbitmq:3.13-management`, плагин `rabbitmq_prometheus` через файл `rabbitmq/enabled_plugins`: `[rabbitmq_management,rabbitmq_prometheus].` Метрики на `:15692/metrics`. Опционально `prometheus.return_per_object_metrics = true` в `rabbitmq.conf` для меток по очередям |
| Конфиг через env | `DB_URL`, `DB_USER`, `DB_PASSWORD`, `RABBIT_HOST`, `RABBIT_USER`, `RABBIT_PASSWORD`, `PROCESSING_DELAY_MS` (worker), `API_URL` (exporter) |
| Spring Boot 4 нюансы | Jackson 3 (`tools.jackson.*`), стартер `spring-boot-starter-webmvc`, `RestClient` уже в webmvc |

### docker-compose: сервисы

| Сервис | Образ | Порт |
|---|---|---|
| postgres | postgres:16 | 5432 |
| rabbitmq | rabbitmq:3.13-management | 5672, 15672, 15692 |
| api | ./api/Dockerfile | 8080 |
| worker | ./worker/Dockerfile | 9101 |
| exporter | ./exporter/Dockerfile | 9102 |
| prometheus | prom/prometheus | 9090 |
| victoriametrics | victoriametrics/victoria-metrics | 8428 |
| node-exporter | prom/node-exporter | 9100 |
| cadvisor | gcr.io/cadvisor/cadvisor | 8081 (внутри 8080) |
| postgres-exporter | prometheuscommunity/postgres-exporter | 9187 |

Prometheus targets: `api:8080`, `worker:9101`, `exporter:9102`, `node-exporter:9100`, `cadvisor:8080`,
`postgres-exporter:9187`, `rabbitmq:15692`. `remote_write: url: http://victoriametrics:8428/api/v1/write`.

Docker Desktop на Windows: node_exporter покажет Linux-VM Docker Desktop, а не Windows — для сдачи ок.
cAdvisor нужны монты `/var/run/docker.sock`, `/sys`, `/var/lib/docker`, `/` (ro).

### Черновик PromQL (для README)

1. `sum(rate(http_requests_total[1m]))` — RPS API.
2. `histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{route="/orders",method="POST"}[5m])))` — p95 POST /orders.
3. `container_memory_working_set_bytes{name="api"}` — память контейнера API.
4. `rabbitmq_queue_messages_ready` — сообщений ждут обработки.
5. `shop_stats_orders_processing` — заказов в PROCESSING.
6. `shop_stats_seconds_since_last_processed` — секунд с последней обработки.
7. `pg_stat_database_numbackends{datname="orders"}` — соединений к PostgreSQL.

---

## Текст задания (оригинал)

**Цель:** написать небольшое приложение и самостоятельно настроить сбор метрик приложения, PostgreSQL, RabbitMQ, хоста и контейнеров.

### 1. Приложение, которое нужно написать

Предметная область — обработка заказов. Язык и веб-фреймворк можно выбрать самостоятельно. Бизнес-логика должна быть простой: приложение нужно прежде всего как стенд для работы с наблюдаемостью.

```
Клиент
  ↓ HTTP
API ─────────→ PostgreSQL
  │
  └──────────→ RabbitMQ ─────────→ Worker
```

API — это ваше основное веб-приложение. Именно к нему пользователь обращается по HTTP. Worker — второй процесс вашего приложения; пользователь напрямую к нему не обращается.

**Как проходит один заказ**

1. Клиент отправляет POST /orders в API.
2. API создаёт запись в PostgreSQL со статусом NEW.
3. API отправляет в RabbitMQ сообщение с order_id созданного заказа.
4. Worker получает сообщение, меняет статус заказа на PROCESSING, ждёт заданное время обработки и затем меняет статус на DONE.
5. Клиент может получить один заказ или список всех заказов через API.

**Что такое Worker**

Worker — отдельный фоновый процесс. Он не принимает пользовательские HTTP-запросы. Worker постоянно читает сообщения из RabbitMQ. Получив order_id, он находит заказ в PostgreSQL, переводит его в PROCESSING, имитирует обработку и после завершения переводит заказ в DONE.

**Какие данные хранит заказ**

| Поле | Что означает | Пример |
|---|---|---|
| id | Идентификатор заказа. Создаётся сервером. | 42 |
| amount | Сумма заказа. Положительное число. | 1500.00 |
| items_count | Количество позиций или товаров в заказе. Положительное целое число. | 3 |
| status | Текущее состояние: NEW, PROCESSING или DONE. | NEW |
| created_at | Время создания заказа. Заполняется API. | 2026-09-04T18:30:00Z |
| processed_at | Время завершения обработки. До статуса DONE может быть пустым; заполняется Worker. | 2026-09-04T18:30:04Z |

created_at и processed_at пригодятся в следующих практиках: по ним можно будет посчитать полное время обработки заказа от создания до DONE.

Пример создания заказа:

```http
POST /orders
Content-Type: application/json

{
  "amount": 1500.00,
  "items_count": 3
}
```

**Какие HTTP-ручки у кого должны быть**

Разделите пользовательские и технические ручки. Пользователь работает только с заказами. Prometheus и собственный exporter работают с техническими ручками.

| Компонент | Ручка | Кто вызывает | Зачем |
|---|---|---|---|
| API | POST /orders | Пользователь/клиент | Создать заказ. |
| API | GET /orders/{id} | Пользователь/клиент | Получить один заказ. |
| API | GET /orders | Пользователь/клиент | Получить все созданные заказы. |
| API | GET /health | Проверка доступности | Проверить, что API запущено. |
| API | GET /metrics | Prometheus | Забрать метрики, которые API считает прямо в своём коде. |
| API | GET /internal/stats | Собственный exporter | Получить обычный JSON со статистикой заказов. Prometheus напрямую сюда не ходит. |
| Worker | GET /metrics | Prometheus | Забрать метрику времени обработки, которую считает Worker. |
| Собственный exporter | GET /metrics | Prometheus | Забрать метрики, которые exporter создал из /internal/stats. |

Порты можно выбрать самостоятельно. Например: API :8080, техническая ручка Worker :9101, собственный exporter :9102. Важно не число порта, а чтобы было понятно, какой компонент и какую ручку предоставляет.

**Что должен возвращать GET /internal/stats**

Эта ручка API нужна только для задания с собственным exporter. Она возвращает обычный JSON. Это ещё не Prometheus-метрики.

```json
{
  "orders_new": 2,
  "orders_processing": 4,
  "orders_done": 120,
  "last_processed_at": "2026-09-04T18:30:10Z"
}
```

| Поле JSON | Откуда взять | Что означает |
|---|---|---|
| orders_new | Посчитать в PostgreSQL записи со статусом NEW. | Сколько заказов ждут обработки. |
| orders_processing | Посчитать записи со статусом PROCESSING. | Сколько заказов сейчас обрабатываются. |
| orders_done | Посчитать записи со статусом DONE. | Сколько заказов уже завершено. |
| last_processed_at | Взять максимальное processed_at среди завершённых заказов. | Когда последний заказ перешёл в DONE. |

Для следующих лабораторных. Добавьте переменную PROCESSING_DELAY_MS. Позже мы специально увеличим задержку Worker и будем наблюдать, как растут очередь RabbitMQ и полное время обработки заказа.

### 2. Что нужно развернуть в Docker

- API;
- Worker;
- PostgreSQL;
- RabbitMQ;
- Prometheus;
- VictoriaMetrics;
- node_exporter;
- cAdvisor;
- postgres_exporter;
- rabbitmq_prometheus exporter;
- собственный exporter для GET /internal/stats.

### 3. Откуда появятся метрики

Главное правило: Prometheus не создаёт исходные метрики за приложение. Он приходит на специальные ручки /metrics и забирает уже подготовленные значения.

```
Пользователь → /orders → API
Prometheus → /metrics API
Prometheus → /metrics Worker
Prometheus → /metrics node_exporter / cAdvisor / postgres_exporter / RabbitMQ
Prometheus → /metrics собственного exporter-а
Собственный exporter → /internal/stats API
```

Почему здесь два разных подхода — /metrics приложения и отдельный exporter? Потому что это две типовые ситуации из реальной работы. Если вы можете менять код сервиса, метрики обычно добавляют прямо в код. Если система не умеет отдавать Prometheus-метрики, но предоставляет данные через API/JSON, для неё пишут exporter-адаптер.

Чтобы не смешивать примеры, в этой лабораторной они разделены специально: HTTP-метрики и время обработки заказа считаются внутри API/Worker. Текущие состояния NEW/PROCESSING/DONE через /metrics API не выводите — их будет создавать только собственный exporter из /internal/stats.

#### 3.1. Метрики, которые считает API

Здесь отдельного exporter нет. Вы подключаете Prometheus client library прямо в код API. Когда API обрабатывает пользовательские запросы, оно обновляет свои метрики. Prometheus затем забирает эти значения через GET /metrics API.

| Метрика API | Когда обновляется | Что показывает |
|---|---|---|
| http_requests_total (Counter) | После каждого HTTP-запроса увеличивается на 1. | Сколько запросов обработало API. Метки: method, route, status. |
| http_request_duration_seconds (Histogram) | После каждого HTTP-запроса записывается его длительность. | Распределение времени ответа; позже можно считать p50/p95/p99. |
| shop_orders_created_total (Counter) | После успешного POST /orders увеличивается на 1. | Сколько заказов создано с момента запуска API. |

Пример: 10 вызовов POST /orders и 5 вызовов GET /orders дадут 15 новых HTTP-запросов, но shop_orders_created_total увеличится только на 10.

#### 3.2. Метрика, которую считает Worker

Здесь отдельного exporter тоже нет. Prometheus client library подключается прямо в Worker. Worker измеряет длительность своей обработки и отдаёт результат через собственный GET /metrics.

| Метрика Worker | Когда обновляется | Что показывает |
|---|---|---|
| shop_order_processing_duration_seconds (Histogram) | Когда Worker завершил обработку заказа. | Сколько секунд прошло от получения сообщения из RabbitMQ до перевода заказа в DONE. |

То есть 3.1 и 3.2 — это один способ: мы владеем кодом API/Worker и встраиваем метрики прямо в него.

#### 3.3. Готовые метрики инфраструктуры

Здесь свой код писать не нужно. Нужно запустить готовые источники метрик и добавить их в Prometheus.

| Что наблюдаем | Что подключить | Какие метрики найти |
|---|---|---|
| Хост | node_exporter | node_cpu_seconds_total; node_memory_MemAvailable_bytes; node_filesystem_avail_bytes. |
| Контейнеры | cAdvisor | container_cpu_usage_seconds_total; container_memory_working_set_bytes; сетевой ввод/вывод. |
| PostgreSQL | postgres_exporter | pg_up; pg_stat_database_numbackends; метрика активности/транзакций. |
| RabbitMQ | rabbitmq_prometheus или exporter | rabbitmq_queue_messages_ready; rabbitmq_queue_messages_unacked; rabbitmq_queue_consumers; rabbitmq_connections. |

#### 3.4. Метрики собственного exporter

Вот здесь способ другой. Мы делаем вид, что API не умеет отдавать нужные бизнес-показатели в формате Prometheus, но умеет показать их обычным JSON через /internal/stats. Поэтому пишем отдельный сервис-переводчик.

1. Exporter вызывает API: GET /internal/stats
2. API возвращает JSON
3. Exporter превращает поля JSON в Prometheus-метрики
4. Prometheus вызывает GET /metrics у exporter-а

| Поле из /internal/stats | Что делает exporter | Что отдаёт в /metrics |
|---|---|---|
| orders_new | Берёт текущее число. | shop_stats_orders_new (Gauge) |
| orders_processing | Берёт текущее число. | shop_stats_orders_processing (Gauge) |
| orders_done | Берёт текущее число. | shop_stats_orders_done (Gauge) |
| last_processed_at | Считает: текущее время − last_processed_at. | shop_stats_seconds_since_last_processed (Gauge) |

Коротко про отличие 3.1–3.2 от 3.4: в 3.1–3.2 метрика рождается внутри кода API/Worker в момент выполнения операции. В 3.4 API отдаёт только обычные данные, а Prometheus-метрику создаёт уже отдельный exporter.

Не дублируйте метрики. http_requests_total, http_request_duration_seconds, shop_orders_created_total и shop_order_processing_duration_seconds относятся только к встроенным метрикам API/Worker. orders_new, orders_processing, orders_done и seconds_since_last_processed в этой работе относятся только к собственному exporter.

### 4. Настройка Prometheus и VictoriaMetrics

- Добавьте в Prometheus цели для API, Worker, node_exporter, cAdvisor, postgres_exporter, RabbitMQ и собственного exporter.
- Откройте Prometheus → Status → Targets. Все работающие цели должны иметь состояние UP.
- Настройте remote_write из Prometheus в VictoriaMetrics.
- Проверьте, что накопленные метрики доступны через VictoriaMetrics.
- Подготовьте 5–7 PromQL-запросов.

Примеры вопросов для PromQL:

- Сколько запросов в секунду получает API?
- Какой p95 времени ответа POST /orders?
- Сколько памяти использует контейнер API?
- Сколько сообщений сейчас ждут обработки в RabbitMQ?
- Сколько заказов сейчас находятся в PROCESSING?
- Сколько секунд прошло с момента последнего обработанного заказа?
- Сколько соединений открыто к PostgreSQL?

### 5. Что сдаётся

1. Репозиторий с кодом API, Worker и собственного exporter.
2. Dockerfile для своих компонентов и docker-compose.yml для запуска всего стенда.
3. Конфигурация Prometheus и remote_write в VictoriaMetrics.
4. Подключённые node_exporter, cAdvisor, postgres_exporter и метрики RabbitMQ.
5. Скриншот Prometheus → Targets со всеми необходимыми целями в состоянии UP.
6. Отдельный список: метрики API; метрика Worker; метрики собственного exporter.
7. 5–7 PromQL-запросов с пояснением, на какой вопрос отвечает каждый.
8. Короткий вывод: чем отличаются встроенные метрики API/Worker, готовые метрики инфраструктуры и метрики собственного exporter.
