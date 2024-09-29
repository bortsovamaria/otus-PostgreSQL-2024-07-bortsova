1. Создаем ВМ/докер c ПГ.

          bortsova@otus-ubuntu:~$ pg_lsclusters
          Ver Cluster Port Status Owner Data directory Log file
          bortsova@otus-ubuntu:~$
          bortsova@otus-ubuntu:~$ sudo pg_createcluster 15 main --start
          [sudo] password for user:
          Creating new PostgreSQL cluster 15/main ...
          /usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/main --auth-local peer --auth-host scram-sha-256 --no-instructions
          The files belonging to this database system will be owned by user "postgres".
          This user must also own the server process.
        
          The database cluster will be initialized with locale "C.UTF-8".
          The default database encoding has accordingly been set to "UTF8".
          The default text search configuration will be set to "english".
        
          Data page checksums are disabled.
        
          fixing permissions on existing directory /var/lib/postgresql/15/main ... ok
          creating subdirectories ... ok
          selecting dynamic shared memory implementation ... posix
          selecting default max_connections ... 100
          selecting default shared_buffers ... 128MB
          selecting default time zone ... Etc/UTC
          creating configuration files ... ok
          running bootstrap script ... ok
          performing post-bootstrap initialization ... ok
          syncing data to disk ... ok
          Ver Cluster Port Status Owner    Data directory              Log file
          15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
          bortsova@otus-ubuntu:~$

2. Создаем БД, схему и в ней таблицу.
    
           bortsova@otus-ubuntu:~$ sudo -u postgres psql -c 'create database db_backup;'
           CREATE DATABASE
           bortsova@otus-ubuntu:~$
           bortsova@otus-ubuntu:~$ sudo -u postgres psql -c '\l'
                                                        List of databases
              Name    |  Owner   | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider |   Access privileges
           -----------+----------+----------+---------+---------+------------+-----------------+-----------------------
            db_backup | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
            postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            |
            template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
                      |          |          |         |         |            |                 | postgres=CTc/postgres
            template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres          +
                      |          |          |         |         |            |                 | postgres=CTc/postgres
           (4 rows)
    
           bortsova@otus-ubuntu:~$
           bortsova@otus-ubuntu:~$ sudo -u postgres psql -d db_backup -c 'create schema schm_backup;'
           CREATE SCHEMA
           bortsova@otus-ubuntu:~$
           bortsova@otus-ubuntu:~$ sudo -u postgres psql -d db_backup -c '\dn'
                    List of schemas
               Name     |       Owner
           -------------+-------------------
            public      | pg_database_owner
            schm_backup | postgres
           (2 rows)
    
           bortsova@otus-ubuntu:~$
           bortsova@otus-ubuntu:~$ sudo -u postgres psql -d db_backup -c 'create table schm_backup.tbl1(c1 int);'
           CREATE TABLE
           bortsova@otus-ubuntu:~$
           bortsova@otus-ubuntu:~$ sudo -u postgres psql -d db_backup -c '\dt'
           Did not find any relations.
           bortsova@otus-ubuntu:~$
           bortsova@otus-ubuntu:~$ sudo -u postgres psql -d db_backup -c '\dt schm_backup.*'
                      List of relations
              Schema    | Name | Type  |  Owner
           -------------+------+-------+----------
            schm_backup | tbl1 | table | postgres
           (1 row)
    
           bortsova@otus-ubuntu:~$

3. Заполним таблицы автосгенерированными 100 записями.

           bortsova@otus-ubuntu:~$ sudo -u postgres psql -d db_backup -c 'INSERT INTO schm_backup.tbl1(c1) SELECT generate_series(1,100);'
           INSERT 0 100
           bortsova@otus-ubuntu:~$
           bortsova@otus-ubuntu:~$ sudo -u postgres psql -d db_backup -c 'SELECT count(*) from schm_backup.tbl1;'
            count 
           -------
              100
           (1 row)
        
           bortsova@otus-ubuntu:~$ 

4. Под линукс пользователем Postgres создадим каталог для бэкапов
   
           bortsova@otus-ubuntu:~$ sudo su - postgres
           postgres@otus:~$
           postgres@otus:~$ cd ~
           postgres@otus:~$
           postgres@otus:~$ pwd
           /var/lib/postgresql
           postgres@otus:~$ mkdir backups
           postgres@otus:~$ ls
           15  backups
           postgres@otus:~$

5. Сделаем логический бэкап используя утилиту COPY
      
           postgres@otus:~$ psql -d db_backup
           psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1))
           Type "help" for help.
        
           db_backup=# \copy schm_backup.tbl1 to '/var/lib/postgresql/backups/tbl1.sql';
           COPY 100
           db_backup=#

6. Восстановим в 2 таблицу данные из бэкапа.
        
        db_backup=#
        db_backup=# create table schm_backup.tbl2(c1 int);
        CREATE TABLE
        db_backup=#
        db_backup=# \copy schm_backup.tbl2 from '/var/lib/postgresql/backups/tbl1.sql';
        COPY 100
        db_backup=#
        db_backup=# select count(*) from schm_backup.tbl2;
         count
        -------
           100
        (1 row)
        
        db_backup=#

7. Используя утилиту pg_dump создадим бэкап с оглавлением в кастомном сжатом формате 2 таблиц

        postgres@otus:~/backups$ pwd
        /var/lib/postgresql/backups
        postgres@otus:~/backups$ pg_dump -d db_backup --schema-only -Fc > db_backup_all.gz
        postgres@otus:~/backups$
        postgres@otus:~/backups$ ls -ll
        total 8
        -rw-rw-r-- 1 postgres postgres 1500 May 24 22:18 db_backup_all.gz
        -rw-rw-r-- 1 postgres postgres  292 May 24 21:51 tbl1.sql
        postgres@otus:~/backups$

8. Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!

        postgres@otus:~/backups$ echo "CREATE DATABASE otus;" | psql -U postgres
        CREATE DATABASE
        postgres@otus:~/backups$ echo "CREATE SCHEMA schm_backup;" | psql -U postgres -d otus
        CREATE SCHEMA
        postgres@otus:~/backups$
        postgres@otus:~/backups$ pg_restore -d otus -U postgres -n schm_backup -t tbl2  '/var/lib/postgresql/backups/db_backup_all.gz'
        postgres@otus:~/backups$
        postgres@otus:~/backups$ psql -d otus -c '\dt schm_backup.*'
                   List of relations
           Schema    | Name | Type  |  Owner
        -------------+------+-------+----------
         schm_backup | tbl2 | table | postgres
        (1 row)
        
        postgres@otus:~/backups$ psql -d otus -c 'select * from schm_backup.tbl2;'
         c1
        ----
        (0 rows)
        
        postgres@otus:~/backups$