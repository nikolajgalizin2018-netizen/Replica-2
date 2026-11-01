# Отчет о выполнении доманего задания на тему: "Репликация и масштабирование. Часть 2"
# Выполнил: Галицин Николай
## Содержание
- [Задание 1. Преимущества методов масштабирования](###задание-1-преимущества-методов-масштабирования)
- [Задание 2. План горизонтального и вертикального шардинга](###задание-2-план-горизонтального-и-вертикального-шардинга)
- [Задание 3*. Настройка шардинга (Citus + Docker)](###задание-3-настройка-шардинга-citus--docker)
### Задание 1
Опишите основные преимущества использования масштабирования методами:

активный master-сервер и пассивный репликационный slave-сервер;
master-сервер и несколько slave-серверов;
Дайте ответ в свободной форме.

### Ответ: 
1. Активный master-сервер и пассивный репликационный slave-сервер
Это схема «горячего резерва»: master обрабатывает всю нагрузку, slave постоянно получает изменения и готов в любой момент заменить master.
Преимущества:

Высокая доступность при отказе. 

Простота. 

Резервное копирование без нагрузки.

Предсказуемость. Все данные в одном месте (на master), slave — точная копия.

2. Master-сервер и несколько slave-серверов
Это схема масштабирования чтения: master принимает запись, несколько slave обслуживают SELECT-запросы.

Преимущества:

Масштабирование чтения.

Распределение нагрузки. Запросы на чтение раскидываются по нескольким серверам, master разгружается.

Отказоустойчивость. При падении одного slave остальные продолжают работать; при падении master один из slave можно повысить.

Гибкость. Можно выделить отдельные slave под аналитику, отчёты или бэкапы, не мешая основному трафику.


### Задание 2
Разработайте план для выполнения горизонтального и вертикального шаринга базы данных. База данных состоит из трёх таблиц:

пользователи,
книги,
магазины (столбцы произвольно).
Опишите принципы построения системы и их разграничение или разбивку между базами данных.

Пришлите блоксхему, где и что будет располагаться. Опишите, в каких режимах будут работать сервера.

### Ответ: План горизонтального и вертикального шардинга
Исходная схема
users (пользователи): user_id, email, name, created_at

books (книги): book_id, title, author, price, description

shops (магазины): shop_id, name, address, rating

Принципы разбивки
Горизонтальный шардинг (по строкам):

users → шардируем по user_id. Хеш от user_id даёт равномерное распределение и быстрый поиск по ID.

books → шардируем по book_id. Книги — независимая сущность, распределяются хешем.

shops → не шардируем. Магазинов мало, они нужны целиком (списки, фильтры). Делаем reference-таблицу, реплицируемую на все шарды.

Вертикальный шардинг (по колонкам):

books → выносим description (тяжёлый текст) в отдельную таблицу book_descriptions на отдельный сервер. Основная таблица books остаётся лёгкой для быстрых запросов.

users → при необходимости выносим редко используемые поля (настройки, историю) в отдельные таблицы.

Блок-схема размещения
```text
┌─────────────────────────────────────────────────────────────┐
│                    COORDINATOR (PostgreSQL + Citus)          │
│  Принимает запросы, маршрутизирует на нужные шарды           │
│  Хранит: shops (reference table, реплицируется на все шарды) │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│   SHARD 1     │     │   SHARD 2     │     │   SHARD 3     │
│               │     │               │     │               │
│ users         │     │ users         │     │ users         │
│ (user_id     │     │ (user_id     │     │ (user_id     │
│  hash 0-33%) │     │  hash 34-66%)│     │  hash 67-100%)│
│               │     │               │     │               │
│ books         │     │ books         │     │ books         │
│ (book_id     │     │ (book_id     │     │ (book_id     │
│  hash ...)   │     │  hash ...)   │     │  hash ...)   │
│               │     │               │     │               │
│ shops         │     │ shops         │     │ shops         │
│ (reference)   │     │ (reference)   │     │ (reference)   │
└───────────────┘     └───────────────┘     └───────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │  VERTICAL SHARD   │
                    │  (отдельный сервер)│
                    │                   │
                    │ book_descriptions │
                    │ (book_id, text)   │
                    └───────────────────┘
```
Режимы работы серверов
Компонент	Режим	Роль
Coordinator	Active	Маршрутизация, хранение reference-таблиц, агрегация результатов
Shard 1–3	Active (worker nodes)	Хранение шардов users и books, параллельные запросы
Vertical Shard	Active (отдельный сервер)	Хранение тяжёлых полей (book_descriptions), запросы по необходимости
Почему shops — reference-таблица: магазинов мало, они нужны почти в каждом запросе, дублирование дешевле cross-shard join.

Почему description вынесен вертикально: текстовые поля занимают много места, редко используются в списках, но нужны в карточке книги. Вынос уменьшает размер основной таблицы и ускоряет типовые запросы.

### Задание 3*
Выполните настройку выбранных методов шардинга из задания 2.

Пришлите конфиг Docker и SQL скрипт с командами для базы данных.

### Ответ:
Для реализации плана используем Citus — расширение PostgreSQL для распределённой БД с шардированием и параллельными запросами.

### Docker Compose конфигурация

```yaml
version: '3.8'

networks:
  citus-net:
    driver: bridge

services:
  coordinator:
    image: citusdata/citus:12.1
    container_name: citus-coordinator
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: shopdb
    volumes:
      - coordinator_data:/var/lib/postgresql/data
    command: |
      -c shared_preload_libraries='citus'
      -c citus.shard_count=32
      -c citus.shard_replication_factor=1
    networks:
      - citus-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 10

  worker1:
    image: citusdata/citus:12.1
    container_name: citus-worker1
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - worker1_data:/var/lib/postgresql/data
    command: |
      -c shared_preload_libraries='citus'
    networks:
      - citus-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 10

  worker2:
    image: citusdata/citus:12.1
    container_name: citus-worker2
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - worker2_data:/var/lib/postgresql/data
    command: |
      -c shared_preload_libraries='citus'
    networks:
      - citus-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 10

volumes:
  coordinator_data:
  worker1_data:
  worker2_data:
```

### SQL-скрипт

```sql
CREATE EXTENSION IF NOT EXISTS citus;

SELECT citus_add_node('citus-worker1', 5432);
SELECT citus_add_node('citus-worker2', 5432);

SELECT * FROM citus_get_active_worker_nodes();

CREATE TABLE shops (
    shop_id   SERIAL PRIMARY KEY,
    name      VARCHAR(255) NOT NULL,
    address   TEXT,
    rating    NUMERIC(2,1)
);

CREATE TABLE users (
    user_id    BIGSERIAL,
    email      VARCHAR(255) NOT NULL,
    name       VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (user_id)
);

CREATE TABLE books (
    book_id   BIGSERIAL,
    title     VARCHAR(255) NOT NULL,
    author    VARCHAR(255),
    price     NUMERIC(10,2),
    shop_id   INT REFERENCES shops(shop_id),
    PRIMARY KEY (book_id)
);

CREATE TABLE book_descriptions (
    book_id     BIGINT PRIMARY KEY,
    description TEXT
);

SELECT create_reference_table('shops');
SELECT create_distributed_table('users', 'user_id');
SELECT create_distributed_table('books', 'book_id');
SELECT create_distributed_table('book_descriptions', 'book_id',
    colocate_with => 'books');

SELECT logicalrelid, distributemethod, colocationid
FROM pg_dist_partition;

SELECT
    s.shardid,
    s.logicalrelid,
    n.nodename,
    n.nodeport
FROM pg_dist_shard s
JOIN pg_dist_placement p ON p.shardid = s.shardid
JOIN pg_dist_node n ON n.groupid = p.groupid
WHERE s.logicalrelid IN ('users'::regclass, 'books'::regclass)
ORDER BY s.logicalrelid, s.shardid
LIMIT 10;
```
### Скрипт включает Citus, подключает двух воркеров, создаёт четыре таблицы и раскладывает их: shops — копия везде, users и books — по шардам, book_descriptions — рядом с books. В конце проверяет, что всё разложилось правильно.

### Ключевые принципы Citus

1. **Coordinator** — точка входа для приложения. Принимает SQL, маршрутизирует на шарды, собирает результаты.
2. **Worker nodes** — хранят шарды и выполняют параллельные запросы.
3. **Distribution column** — ключ шардирования. Для `users` — `user_id`, для `books` — `book_id`. Ключ шардинга должен быть частью PRIMARY KEY.
4. **Reference table** (`shops`) — реплицируется на все воркеры, join с ней локальный и быстрый.
5. **Colocation** (`book_descriptions` и `books`) — связанные таблицы с одинаковым ключом шардирования попадают на одни воркеры, join локальный.

