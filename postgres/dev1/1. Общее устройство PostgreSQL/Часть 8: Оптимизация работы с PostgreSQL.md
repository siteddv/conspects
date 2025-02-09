Часть 8: Оптимизация работы с PostgreSQL

1. Введение

PostgreSQL – мощная, но требовательная к ресурсам СУБД. Если база данных растёт, а нагрузка увеличивается, могут возникнуть проблемы с производительностью. Для их решения PostgreSQL предоставляет:

Оптимизацию соединений (PgBouncer, пул соединений).

Эффективное использование памяти и кеширования.

Работу с индексами.

Параллельное выполнение запросов.

Диагностику медленных запросов и анализ производительности.



---

2. Ограничения на количество соединений

PostgreSQL использует многопроцессную архитектуру: каждое соединение создаёт отдельный процесс. Если одновременно работает 1000 клиентов, база данных должна управлять 1000 процессами, что создаёт нагрузку на CPU и память.

Настройки максимального количества соединений

SHOW max_connections;
ALTER SYSTEM SET max_connections = 500;

Однако, увеличение max_connections не всегда решает проблему, так как каждое соединение потребляет ресурсы.

Решение – пул соединений (PgBouncer).


---

3. Использование пулеров соединений

PgBouncer – это промежуточный сервер, который принимает запросы от клиентов и переиспользует соединения к PostgreSQL.

Преимущества PgBouncer:

Меньшая нагрузка на сервер – PostgreSQL обрабатывает не все соединения, а только активные.

Стабильность – уменьшает количество "тяжёлых" процессов.

Повышение производительности – запросы обрабатываются быстрее.


Настройка PgBouncer

1. Установка:

sudo apt install pgbouncer


2. Настройка pgbouncer.ini

[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0
max_client_conn = 1000
default_pool_size = 50


3. Запуск PgBouncer:

sudo systemctl start pgbouncer


4. Подключение через PgBouncer:

psql -h 127.0.0.1 -p 6432 -U myuser -d mydb



Диагностика соединений

SHOW POOLS;

Эта команда показывает активные и свободные соединения в пуле.


---

4. Оптимизация работы с памятью

PostgreSQL использует несколько уровней кеширования, которые помогают ускорять запросы.

4.1 Основные параметры памяти

Проверка текущих значений:

SHOW shared_buffers;
SHOW work_mem;
SHOW maintenance_work_mem;

Рекомендации:

ALTER SYSTEM SET shared_buffers = '8GB';
ALTER SYSTEM SET work_mem = '64MB';
ALTER SYSTEM SET maintenance_work_mem = '512MB';

4.2 Как работает кеширование

Когда выполняется запрос:

1. PostgreSQL ищет данные в буфере (shared_buffers).


2. Если данных нет – запрашивает их у операционной системы (Page Cache).


3. Если и там нет – идёт на диск.



Как проверить использование кеша?

EXPLAIN ANALYZE SELECT * FROM orders WHERE id = 100;

Если в EXPLAIN указано Shared Hit, значит, данные взяты из кеша.


---

5. Оптимизация запросов с индексами

5.1 Виды индексов

Создание индекса:

CREATE INDEX idx_users_name ON users(name);

Проверка использования индекса:

EXPLAIN ANALYZE SELECT * FROM users WHERE name = 'Alice';

Когда индексы НЕ работают?

1. Функции в WHERE не используют индекс

SELECT * FROM users WHERE LOWER(name) = 'alice';  -- НЕ ИСПОЛЬЗУЕТ индекс

Решение – Функциональный индекс:

CREATE INDEX idx_users_lower_name ON users(LOWER(name));


2. LIKE '%word%' не использует B-Tree

Используйте GIN-индекс:


CREATE INDEX idx_users_gin_name ON users USING gin(to_tsvector('english', name));




---

6. Параллельное выполнение запросов

PostgreSQL поддерживает многопоточное выполнение запросов, чтобы ускорить сложные выборки.

Пример:

EXPLAIN ANALYZE SELECT COUNT(*) FROM orders;

Вывод:

Parallel Seq Scan on orders (workers: 4)

PostgreSQL разделил запрос на 4 потока.

Настройки:

SHOW max_parallel_workers_per_gather;  -- Количество потоков
SHOW parallel_tuple_cost;  -- Порог стоимости параллелизации

Рекомендации:

max_parallel_workers_per_gather = 4-8

parallel_tuple_cost = 0.1


Принудительное включение параллельности:

SET parallel_tuple_cost = 0.01;


---

7. Анализ и диагностика производительности

Для поиска медленных запросов используйте pg_stat_statements.

7.1 Включение модуля

Добавьте в postgresql.conf:

shared_preload_libraries = 'pg_stat_statements'

Перезапустите сервер и создайте расширение:

CREATE EXTENSION pg_stat_statements;

7.2 Поиск медленных запросов

SELECT query, calls, total_time, mean_time
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;

7.3 Как ускорить медленные запросы?

Добавить индексы (CREATE INDEX).

Увеличить work_mem.

Разбить запрос на меньшие части.

Использовать EXPLAIN ANALYZE.



---

8. Итоги восьмой части

PgBouncer снижает нагрузку, переиспользуя соединения.

Настройки памяти (shared_buffers, work_mem) ускоряют кеширование.

Индексы ускоряют поиск, но важно правильно их применять.

Параллельные запросы ускоряют COUNT(), JOIN, GROUP BY.

pg_stat_statements помогает анализировать медленные запросы.



---

Что дальше?

Следующая часть будет посвящена расширяемости PostgreSQL: пользовательские типы данных, функции, триггеры и плагины. Готов продолжить?

