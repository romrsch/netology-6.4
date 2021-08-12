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


Восстановим бэкап БД в `test_database`
```
psql -U postgres -f /tmp/test_dump.sql  test_database
```

![альт](https://i.ibb.co/RCvTVZ5/Screenshot-10.jpg)

```
psql -h localhost -p 5432 -U postgres -W
postgres=# \c test_database
```
![альт](https://i.ibb.co/ftWjf5g/Screenshot-11.jpg)

```
test_database=# ANALYZE VERBOSE public.orders;

test_database=# select avg_width from pg_stats where tablename='orders';
```
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

Нужно преобразовать существующую таблицу, а именно переименовать её.
пересоздадим таблицу заново
переименовывать исходную таблицу и переносить данные в новую

```
test_database=# alter table orders rename to orders_simple;
ALTER TABLE
test_database=# create table orders (id integer, title varchar(80), price integer) partition by range(price);
CREATE TABLE
test_database=# create table orders_2 partition of orders for values from (0) to (499);
CREATE TABLE
test_database=# create table orders_1 partition of orders for values from (499) to (999999999);
CREATE TABLE
test_database=# insert into orders (id, title, price) select * from orders_simple;
INSERT 0 8
test_database=# 
test_database=# 

```
Можно было изначально создать 

---
### Задача 4

Используя утилиту `pg_dump` создайте бекап БД `test_database`.

Как бы вы доработали бэкап-файл, чтобы добавить уникальность значения столбца title для таблиц test_database?



***Ответ:***
```
root@postgresql:/var/lib/postgresql/data# pg_dump -U postgres -d test_database >test_database_dump.sql
```
![альт](https://i.ibb.co/BfYTB4J/Screenshot-14.jpg)
---

