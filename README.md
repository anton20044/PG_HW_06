| **<br/>Лабораторная работа №6 по курсу "PostgreSQL для администраторов баз данных и разработчиков"<br/>"Настройка autovacuum с учетом особенностей производительности"<br/>**|
|---|

<br/>

## Задание:
### * запустить нагрузочный тест pgbench
### * настроить параметры autovacuum
### * проверить работу autovacuum

<br/>

## Решение:

* Создал виртуальную машину, установил Ubuntu 22.04, PostgreSQL v.16
* Создал новый кластер PostgreSQL main2 (sudo pg_createcluster 16 main2)

![pg_lscluster](image/dz06_01.jpg)

* Создал БД dz6 (create database dz6;), инициализировал для нее pgbench и запустил с параметрами (sudo -u postgres pgbench -i dz6 -p 5433)

![pgbench](image/dz06_02.jpg)

* Применил параметры, приложенные к заданию. Перезапустил кластер PostgreSQL и повторил тест

![pgbench](image/dz06_03.jpg)

* После применения параметров, производительность СУБД выросла примерно в 2 раза (156 tps при параметрах по умолчанию и 321 tps после применения параметров). Объянение:
  ** Увеличение min_wal_size с 80Мб до 4Гб повлияло на производительность, т.к. увеличилось число журналов, которые будут не удаляться с диска и создаваться заново, а перезаписываться, что быстрее
  ** Увеличение shared_buffers повлияло на то, что больше страниц оказалось в кэше СУБД и их не пришлось поднимать с диска
  ** Увеличение effective_cache_size повлияло на планировщик, т.к. с большей вероятностью он мог найти страницу в кэше ОС, если в кэше СУБД ее не оказалось
  ** Увеличение maintenance_work_mem повлияло на более быстрое выполнение процессов autovacuum и как следствие мы получили меньший блоатинг таблицы, т.к. мертвые строки стали вычищатся быстрее, освобождая место для записи новых данных
  ** Уличение wal_buffers (с 4 до 16 мб) повлияло на то, что wal файлы стали записываться за один раз (стандартный развер wal 16мб), а не четыре раза (4мб х 4)
* Создал таблицу test в БД dz6 (create table test (word char(80));)
* Заполнил ее сгенерированными данными (insert into test SELECT 'word' || generate_series FROM generate_series(1,1000000);)
* Посмотрел размер таблицы ( SELECT pg_size_pretty(pg_total_relation_size('test'));)

![table_size](image/dz06_04.jpg)

* Обновил данные 5 раз, функция обновления:
```
DO $$DECLARE r record;
BEGIN
    FOR r IN SELECT generate_series FROM generate_series(1,5)
    LOOP
        update test set word = word || r;
    END LOOP;
END$$;
```

* Посмотрел размер таблицы

![table_size1](image/dz06_05.jpg)

* Посмотрел статистику по таблице (количество живых/мертвых строк)
```
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum 
FROM pg_stat_user_TABLEs WHERE relname = 'test';
```

![table_stat](image/dz06_06.jpg)

* Дождался процесса autovacuum и посмотрел статистику по таблице, мертвые строки были вычищены, место, занятое таблицей на диске, осталось без изменения

![table_stat1](image/dz06_07.jpg)

* Еще раз обновил строки таблицы 5 раз (использовался скрипт из примера выше), посмотрел статистику таблицы. Посмотрел ее размер, размер не увеличился, т.к. было достаточно свободного места, чтобы поместить новые данные

![table_stat2](image/dz06_08.jpg)

* Отключил на таблице autovacuum (ALTER TABLE test SET (autovacuum_enabled = off);), обновил строки таблицы 10 раз
```
DO $$DECLARE r record;
BEGIN
    FOR r IN SELECT generate_series FROM generate_series(1,10)
    LOOP
        update test set word = word || r;
    END LOOP;
END$$;
```

![table_stat3](image/dz06_09.jpg)

* Таблица стала занимать на диске в два раза больше места, т.к. процесс autovacuum выключен и теперь никто не очищает страницы с мертвыми строками

![table_stat4](image/dz06_10.jpg)
