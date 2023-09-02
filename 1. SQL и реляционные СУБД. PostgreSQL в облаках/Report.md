# Отчет

Поднимаем контейнер командой:
```bash
docker-compose up -d
```
Подключаемся к контейнеру:

```bash
docker exec -it otus_postgresadvanced-db-1 sh
```
Устанавливаем sudo:
```bash
apt-get update && apt-get install -y sudo
```
Запускаем необходимое количество сессий psql:
```bash
sudo -u postgres psql
```
Выключаем AUTOCOMMIT:
```bash
\set AUTOCOMMIT OFF
```
Создаем таблицу с данными:
```bash
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
```
Выводим текущий уровень изоляции транзакции:
```bash
$ show transaction isolation level;

read committed
```

Начинаем новую транзакцию в обоих сессиях
```bash
BEGIN;
```

В первой сессии добавяем новую запись
```bash
insert into persons(first_name, second_name) values('sergey', 'sergeev');
```
Запрашиваем данные из таблицы `persons` во второй сессии:

```bash
$ select * from persons;

id | first_name | second_name
----+------------+-------------
1 | ivan       | ivanov
2 | petr       | petrov
```

> видите ли вы новую запись и если да то почему?

Новая запись не видна, потому что при уровне изоляции `read committed` мы видим только закомиченные данные и грязное чтение не допускается.

Завершаем первую транзакцию:
```bash
COMMIT;
```

Запрашиваем данные из таблицы `persons` во второй сессии:

```bash
$ select * from persons;

 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
> видите ли вы новую запись и если да то почему?

Новая запись видна, потому что при уровне изоляции `read committed` мы видим только закомиченные данные.

Начинаем новые транзакции с уровнем `repeatable read`:
```bash
set transaction isolation level repeatable read;
```

Добавляем новую запись в первой сессии:
```bash
insert into persons(first_name, second_name) values('sveta', 'svetova');
```


Запрашиваем данные из таблицы `persons` во второй сессии:

```bash
$ select * from persons;

 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

> видите ли вы новую запись и если да то почему?

Новая запись не видна, потому что при уровне изоляции `repeatable read` грязное чтение не допускается.

Завершаем первую транзакцию:
```bash
COMMIT;
```

Запрашиваем данные из таблицы `persons` во второй сессии:

```bash
$ select * from persons;

 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

Новая запись не видна, потому что при уровне изоляции `repeatable read` в PostgreSQL не допускается фантомное чтение.

Завершаем вторую транзакцию:
```bash
COMMIT;
```

Запрашиваем данные из таблицы `persons` во второй сессии:
```bash
select * from persons;

 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

>видите ли вы новую запись и если да то почему?

Новая запись видна, потому что все предыдущие транзакции закомитили свои изменения, а запрос выполняется в новой транзакции.