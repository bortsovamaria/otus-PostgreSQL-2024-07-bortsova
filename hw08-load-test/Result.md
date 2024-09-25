
* развернуть виртуальную машину любым удобным способом и поставить на неё PostgreSQL 15 любым способом
     Устанавливаем PostgreSQL 15
   
         sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15


Проводим измерение нагрузки через pgbench, со значениями по умолчанию для PostgreSQL 15. Параметры запуска pgbench такие:

              bortsova@otus:~$ sudo -u postgres psql
              psql (15.3 (Ubuntu 15.3-1.pgdg20.04+1))
              Type "help" for help.
        
              postgres=# \l
                                                               List of databases
                 Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges   
              -----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
               postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | 
               template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
                         |          |          |             |             |            |                 | postgres=CTc/postgres
               template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
                         |          |          |             |             |            |                 | postgres=CTc/postgres
              (3 rows)
        
              postgres=# create database bench_tests;
              CREATE DATABASE
              postgres=# \q
              bortsova@otus:~$ 
              bortsova@otus:~$ sudo -u postgres pgbench -i bench_tests
              dropping old tables...
              NOTICE:  table "pgbench_accounts" does not exist, skipping
              NOTICE:  table "pgbench_branches" does not exist, skipping
              NOTICE:  table "pgbench_history" does not exist, skipping
              NOTICE:  table "pgbench_tellers" does not exist, skipping
              creating tables...
              generating data (client-side)...
              100000 of 100000 tuples (100%) done (elapsed 1.24 s, remaining 0.00 s)
              vacuuming...
              creating primary keys...
              done in 2.07 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 1.75 s, vacuum 0.04 s, primary keys 0.27 s).
              bortsova@otus:~$ 
              bortsova@otus:~$ sudo -u postgres pgbench --client=20 --connect --jobs=5 --progress=30 --time=300 bench_tests
              pgbench (15.3 (Ubuntu 15.3-1.pgdg20.04+1))
              starting vacuum...end.
              progress: 30.0 s, 207.2 tps, lat 92.675 ms stddev 90.687, 0 failed
              progress: 60.0 s, 225.3 tps, lat 85.105 ms stddev 83.888, 0 failed
              progress: 90.0 s, 216.7 tps, lat 88.646 ms stddev 73.884, 0 failed
              progress: 120.0 s, 206.0 tps, lat 93.796 ms stddev 78.239, 0 failed
              progress: 150.0 s, 218.2 tps, lat 87.970 ms stddev 67.603, 0 failed
              progress: 180.0 s, 231.6 tps, lat 82.635 ms stddev 60.447, 0 failed
              progress: 210.0 s, 183.1 tps, lat 105.927 ms stddev 126.609, 0 failed
              progress: 240.0 s, 222.9 tps, lat 85.975 ms stddev 81.786, 0 failed
              progress: 270.0 s, 184.7 tps, lat 104.719 ms stddev 95.683, 0 failed
              progress: 300.0 s, 217.1 tps, lat 88.755 ms stddev 79.458, 0 failed
              transaction type: <builtin: TPC-B (sort of)>
              scaling factor: 1
              query mode: simple
              number of clients: 20
              number of threads: 5
              maximum number of tries: 1
              duration: 300 s
              number of transactions actually processed: 63410
              number of failed transactions: 0 (0.000%)
              latency average = 91.079 ms
              latency stddev = 84.750 ms
              average connection time = 3.554 ms
              tps = 211.293792 (including reconnection times)
              bortsova@otus:~$ 


* настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины
Берем калькулятор параметров [PGTune](https://pgtune.leopard.in.ua/) и считаем параметры для следующей конфигурации:
    
            # DB Version: 15
            # OS Type: linux
            # DB Type: mixed
            # Total Memory (RAM): 4 GB
            # CPUs num: 2
            # Connections num: 20
            # Data Storage: ssd
     
            max_connections = 20
            shared_buffers = 1GB
            effective_cache_size = 3GB
            maintenance_work_mem = 256MB
            checkpoint_completion_target = 0.9
            wal_buffers = 16MB
            default_statistics_target = 100
            random_page_cost = 1.1
            effective_io_concurrency = 200
            work_mem = 13107kB
            min_wal_size = 1GB
            max_wal_size = 4GB


* Применяем параметры на СУБД PostgreSQL и перезапускаем кластер, чтобы применились параметры:

           bortsova@otus:~$ sudo -u postgres psql
           psql (15.3 (Ubuntu 15.3-1.pgdg20.04+1))
           Type "help" for help.
        
           postgres=# ALTER SYSTEM SET max_connections = '20';
           ALTER SYSTEM
           postgres=# ALTER SYSTEM SET shared_buffers = '1GB';
           ALTER SYSTEM
           postgres=# ALTER SYSTEM SET effective_cache_size = '3GB';
           ALTER SYSTEM
           postgres=# ALTER SYSTEM SET maintenance_work_mem = '256MB';
           ALTER SYSTEM
           postgres=# ALTER SYSTEM SET checkpoint_completion_target = '0.9';
           ALTER SYSTEM
           postgres=# ALTER SYSTEM SET wal_buffers = '16MB';
           ALTER SYSTEM
           postgres=# ALTER SYSTEM SET default_statistics_target = '100';
           ALTER SYSTEM
           postgres=# ALTER SYSTEM SET random_page_cost = '1.1';
           ALTER SYSTEM
           postgres=# ALTER SYSTEM SET effective_io_concurrency = '200';
           ALTER SYSTEM
           postgres=# ALTER SYSTEM SET work_mem = '13107kB';
           ALTER SYSTEM
           postgres=# ALTER SYSTEM SET min_wal_size = '1GB';
           ALTER SYSTEM
           postgres=# ALTER SYSTEM SET max_wal_size = '4GB';
           ALTER SYSTEM
           postgres=#  
           postgres=# alter system set wal_level='minimal';
           ALTER SYSTEM
           postgres=# alter system set max_wal_senders='0';
           ALTER SYSTEM
           postgres=# alter system set synchronous_commit='off';
           ALTER SYSTEM
           postgres=# alter system set fsync='off';
           ALTER SYSTEM
           postgres=# alter system set full_page_writes='off'; 
           ALTER SYSTEM
           postgres=# 
           postgres=# \q
           bortsova@otus:~$ 
           bortsova@otus:~$ sudo pg_ctlcluster 15 main restart
           bortsova@otus:~$ 
           bortsova@otus:~$ pg_lsclusters 
           Ver Cluster Port Status Owner    Data directory              Log file
           15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
           bortsova@otus:~$ 


* нагрузить кластер через утилиту через утилиту pgbench (https://postgrespro.ru/docs/postgrespro/14/pgbench)
Проводим измерение нагрузки через pgbench. Параметры запуска pgbench такие:
    

              bortsova@otus:~$ sudo -u postgres pgbench --client=20 --connect --jobs=5 --progress=30 --time=300 bench_tests
              pgbench (15.3 (Ubuntu 15.3-1.pgdg20.04+1))
              starting vacuum...end.
              pgbench: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  sorry, too many clients already
              pgbench: error: client 2 aborted while establishing connection
              progress: 30.0 s, 322.0 tps, lat 52.931 ms stddev 28.149, 0 failed
              progress: 60.0 s, 323.6 tps, lat 52.094 ms stddev 27.503, 0 failed
              progress: 90.0 s, 328.2 tps, lat 51.326 ms stddev 26.695, 0 failed
              progress: 120.0 s, 323.7 tps, lat 52.067 ms stddev 27.247, 0 failed
              progress: 150.0 s, 326.8 tps, lat 51.639 ms stddev 27.110, 0 failed
              progress: 180.0 s, 324.6 tps, lat 52.092 ms stddev 26.365, 0 failed
              progress: 210.0 s, 324.8 tps, lat 51.968 ms stddev 26.565, 0 failed
              progress: 240.0 s, 325.0 tps, lat 51.929 ms stddev 26.235, 0 failed
              progress: 270.0 s, 321.7 tps, lat 52.542 ms stddev 26.743, 0 failed
              progress: 300.0 s, 325.2 tps, lat 51.913 ms stddev 26.672, 0 failed
              transaction type: <builtin: TPC-B (sort of)>
              scaling factor: 1
              query mode: simple
              number of clients: 20
              number of threads: 5
              maximum number of tries: 1
              duration: 300 s
              number of transactions actually processed: 97383
              number of failed transactions: 0 (0.000%)
              latency average = 52.047 ms
              latency stddev = 26.935 ms
              average connection time = 6.546 ms
              tps = 324.580605 (including reconnection times)
              pgbench: error: Run was aborted; the above results are incomplete.
              bortsova@otus:~$ 


* написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему
Выводы:
    * `number of transactions actually processed` было `63410` стало `97383`. Количество транзакций увеличилось в `1,54` раза и это хороший результат.
    * `latency average` было `91.079 ms` стало `52.047 ms`. Средняя задержка уменьшилась в `1,75` раза и это хороший результат.
    * `latency stddev` было `84.750 ms` стало `26.935 ms`. Средняя задержка дисковых устройств уменьшилась в `3,15` раза и это очень хороший результат.
    * `average connection time` было `3.554 ms` стало `6.546 ms`. Увеличилось среднее время подключения в `1,84` раза, что не есть хорошо. Но это вызывано тем, что pgbench упирался в количество подключений, которое мы уменьшили.
    * `tps` было `211.293792` стало `324.580605`. В `1,54` раза увеличилось количество транзакций в секунду, что есть очень хорошо.
Результат:
    * Производительность стала намного выше, но в ущерб надежности.
    * Второй прогон pgbench выполнился не полностью т.к. уперся в количество подключений [max_connections](https://postgrespro.ru/docs/postgrespro/15/runtime-config-connection#GUC-MAX-CONNECTIONS), которое мы уменьшили с `100` до `20` в процессе настройки.
