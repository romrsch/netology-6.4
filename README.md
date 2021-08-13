# netology-6.4
## Домашнее задание к занятию "6.4. PostgreSQL"

### Задача 1

Используя docker поднимите инстанс PostgreSQL (версию 13). Данные БД сохраните в `volume`.

Подключитесь к БД PostgreSQL используя `psql`.

Воспользуйтесь командой `\?` для вывода подсказки по имеющимся в `psql` управляющим командам.
Найдите и приведите управляющие команды для:

* Ввода списка БД
* Подключения к БД
* Вывода списка таблиц
* Вывода описания содержимого таблиц
* Выхода из psql


***Ответ:***

```
docker login
Login Succeeded

docker pull postgres:13

docker images
postgres     13        b2fcd079c1d4   3 weeks ago   315MB

docker volume create vol_postgres
vol_postgres

docker volume ls
local     vol_postgres
```
Запуск
```
docker run --rm --name postgresql --hostname postgresql  -e POSTGRES_PASSWORD=123456 -ti -p 5432:5432 -v vol_postgres:/var/lib/postgresql/data  postgres:13

```
![альт](https://i.ibb.co/pKktk4D/Screenshot-4.jpg)


Скопируем бэкап БД `test_dump.sql` в контейнер
```
docker cp ../6.4/test_dump.sql  postgresql:/tmp
```
Переходим внутрь работающего контейнера PostreSQL
```
docker exec -it postgresql bash
```

![альт](https://i.ibb.co/vwdMDL9/Screenshot-1.jpg)

Подключаемся:
```
psql -h localhost -p 5432 -U postgres -W
```

![альт](https://i.ibb.co/R3dDQG3/Screenshot-2.jpg)

* Ввод списка БД 
```
  \l[+]   [PATTERN]      list databases
```
![альт](https://i.ibb.co/yss2FyQ/Screenshot-3.jpg)


* Подключения к БД

```
\conninfo              display information about current connection
```

![альт](https://i.ibb.co/B4RF23X/Screenshot-5.jpg)

* Вывод списка таблиц
```
\dt[S+] [PATTERN]      list tables
```
![альт](https://i.ibb.co/MMnKJqG/Screenshot-6.jpg)

* Вывод описания содержимого таблиц
```
  \d[S+]                 list tables, views, and sequences
 ```
![альт](https://i.ibb.co/V3ZpgqH/Screenshot-8.jpg)

* Выход из psql
```
  \q                     quit psql
```
```
postgres=# \q
root@postgresql:/# 
```
---

### Задача 2

Используя `psql` создайте БД `test_database`.

Изучите бэкап БД.

Восстановите бэкап БД в `test_database`.

Перейдите в управляющую консоль psql внутри контейнера.

Подключитесь к восстановленной БД и проведите операцию `ANALYZE` для сбора статистики по таблице.

Используя таблицу `pg_stats`, найдите столбец таблицы orders с наибольшим средним значением размера элементов в байтах.

Приведите в ответе команду, которую вы использовали для вычисления и полученный результат.


***Ответ:***

создаём БД `test_database`

![альт](https://i.ibb.co/zs00jnt/Screenshot-9.jpg)


Восстановим бэкап БД в нашу созданную БД `test_database` и посмотрим список таблиц.
```
psql -U postgres -f /tmp/test_dump.sql  test_database
```

![альт](https://i.ibb.co/RCvTVZ5/Screenshot-10.jpg)


Подключимся к восстановленной БД "test_database"
```
psql -h localhost -p 5432 -U postgres -W
postgres=# \c test_database
```
![альт](https://i.ibb.co/ftWjf5g/Screenshot-11.jpg)


ANALYZE — собрать статистику по базе данных. 
* ANALYZE [ VERBOSE ] [ имя_таблицы ]
```
test_database=# ANALYZE VERBOSE public.orders;
```
Найдём столбец таблицы `orders` с наибольшим средним значением размера элементов в байтах.

Представление `pg_stats` открывает доступ к информации, хранящейся в каталоге pg_statistic.

Применим запрос с параметрами:

* `avg_width` - Средний размер элементов в столбце, в байтах
* `tablename`  - Имя таблицы

```
test_database=# select avg_width from pg_stats where tablename='orders';
```
Скриншот работы команды `ANALYZE` и запроса с `pg_stats`

![альт](https://i.ibb.co/Y3NWLMf/Screenshot-12.jpg)

---
### Задача 3

Архитектор и администратор БД выяснили, что ваша таблица `orders` разрослась до невиданных размеров и поиск по ней занимает долгое время. 
Вам, как успешному выпускнику курсов DevOps в нетологии предложили провести разбиение таблицы на 2 две:
шардировать на: 
* orders_1 - price  > 499
* orders_2 - price <= 499

Предложите SQL-транзакцию для проведения данной операции.

Можно ли было изначально исключить "ручное" разбиение при проектировании таблицы orders?


***Ответ:***

Партиционирование (partitioning) — это разбиение таблиц, содержащих большое количество записей, на логические части по неким выбранным администратором критериям. Партиционирование таблиц делит весь объем операций по обработке данных на несколько независимых и параллельно выполняющихся потоков, что существенно ускоряет работу СУБД. Для правильного конфигурирования параметров партиционирования необходимо, чтобы в каждом потоке было примерно одинаковое количество записей.

* Нужно существующую таблицу `orders` переименовать (например) в `orders_orig` .

* Создать таблицу `orders` заново с `PARTITION BY` (задаёт стратегию секционирования таблицы). Таблица, созданная с этим указанием, называется секционируемой таблицей. Задаваемый в скобках список столбцов или выражений формирует ключ разбиения таблицы.  
* Создаем пустую таблицу `orders_2` с размером от 0 до 499. 
`PARTITION OF таблица_родитель { FOR VALUES указание_границ_секции | DEFAULT } `. 
Создаёт таблицу в виде секции указанной родительской таблицы `orders`. 

* Создаем пустую таблицу `orders_1` с размером от 499 и более.
* Переносим данные из старой таблицы `orders_orig` в новую секционированную таблицу `orders`.

```
test_database=# ALTER TABLE orders RENAME TO orders_orig;
ALTER TABLE
test_database=# CREATE TABLE orders (id integer, title varchar(80), price integer) PARTITION BY RANGE (price);
CREATE TABLE
test_database=# CREATE TABLE orders_2 PARTITION OF orders FOR VALUES FROM (0) TO (499);
CREATE TABLE
test_database=# CREATE TABLE orders_1 PARTITION OF orders FOR VALUES FROM (499) TO (999999999);
CREATE TABLE
test_database=# INSERT INTO orders (id, title, price) SELECT * FROM orders_orig; 
INSERT 0 8

test_database=# 
```
Итого:

![альт](https://i.ibb.co/F3Zq8KJ/Screenshot-16.jpg)

Можно было изначально создать секционированную таблицу с необходимыми значениями, чтобы не разбивать её потом вручную.

---
### Задача 4

Используя утилиту `pg_dump` создайте бекап БД `test_database`.

Как бы вы доработали бэкап-файл, чтобы добавить уникальность значения столбца `title` для таблиц `test_database`?



***Ответ:***

Создаём бэкап БД с помощью утилиты `pg_dump`:
```
root@postgresql:/var/lib/postgresql/data# pg_dump -U postgres -d test_database >test_database_dump.sql
```
![альт](https://i.ibb.co/BfYTB4J/Screenshot-14.jpg)


Для уникальности можно создать индекс:

```
    CREATE INDEX ON orders ((lower(title)));
```

CREATE INDEX создаёт индексы по указанному столбцу(ам) заданного отношения, которым может быть таблица. Индексы применяются в первую очередь для оптимизации производительности базы данных.

Создание индекса по выражению lower(title), позволяющего эффективно выполнять регистронезависимый поиск:
(В этом примере можно опустить имя индекса, чтобы имя выбрала система, например orders_lower_idx.)

---

