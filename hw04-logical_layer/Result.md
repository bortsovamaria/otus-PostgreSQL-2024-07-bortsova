*  создайте новый кластер PostgresSQL 14

        sudo pg_createcluster 14 main_**
        sudo pg_ctlcluster start 14 main_**
        
        sudo pg_lsclusters
         Ver Cluster Port Status Owner    Data directory              Log file
         14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log


* зайдите в созданный кластер под пользователем postgres

      sudo -u postgres psql -U postgres_**
      select current_user;

 * создайте новую базу данных testdb

        create database testdb;_**
        \l
        select * from pg_database;


* зайдите в созданную базу данных под пользователем postgres

      sudo -u postgres psql -d testdb -U postgres_**
      \conninfo 
      select current_user;
      select user;
      select current_database();


* создайте новую схему testnm

      create schema testnm;
      \dn
       select * from pg_namespace;


* создайте новую таблицу t1 с одной колонкой c1 типа integer

      create table t1(c1 int);
      \dt 
      select * from pg_tables;


* вставьте строку со значением c1=1

      insert into t1(c1) values(1);

 * создайте новую роль readonly

        create role readonly;
        \dg 
        \du
        select * from pg_roles;


* дайте новой роли право на подключение к базе данных testdb

      grant connect on database testdb to readonly;
      \l+ testdb 
      select * from pg_database where datname='testdb';


* дайте новой роли право на использование схемы testnm

      grant usage on schema testnm to readonly;
      \dn+ testnm 
      select * from pg_namespace where nspname='testnm';


* дайте новой роли право на select для всех таблиц схемы testnm

      grant select on all tables in schema testnm to readonly;


* создайте пользователя testread с паролем test123

      create user testread with password 'test123';
      \du 
      select * from pg_user;


* дайте роль readonly пользователю testread

      grant readonly to testread;
      \du+ testread 
       select roleid::regrole, member::regrole, grantor::regrole from pg_auth_members where member::regrole::text='testread';


*  зайдите под пользователем testread в базу данных testdb

        sudo -u postgres psql -d testdb -U testread -h 127.0.0.1


* сделайте select * from t1;

      +


* получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)

Нет


* напишите что именно произошло в тексте домашнего задания

       ERROR:  permission denied for table t1

* у вас есть идеи почему? ведь права то дали?
Выдавали роли readonly права на выборку всех таблиц в схеме testnm, а таблица t1 находится в схеме public. 
В данном случае пользователь testread является частью роли(группы) readonly. Но у нас нет наследования прав_**


* посмотрите на список таблиц

Таблица t1 находится в схеме public


* а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)

Могли явно указать имя схемы при создании таблицы


* вернитесь в базу данных testdb под пользователем postgres
      
      sudo -u postgres psql -d testdb -U postgres


* удалите таблицу t1
      
      drop table t1;


* создайте ее заново но уже с явным указанием имени схемы testnm
      
      create table testnm.t1(c1 int);


* вставьте строку со значением c1=1
      
      insert into testnm.t1(c1) values(1);


* зайдите под пользователем testread в базу данных testdb
      
      sudo -u postgres psql -d testdb -U testread -h 127.0.0.1


* сделайте select * from testnm.t1;

        +


* получилось?

       ERROR:  permission denied for table t1


* есть идеи почему? если нет - смотрите шпаргалку

Необходимо выдать явные права на выборку таблицы testnm.t1 пользователю testread_**

     grant select on testnm.t1 to testread;


* как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку

Выдать пользователю testread или роли(группе) readonly права на выборку всех таблиц в схеме testnm. Или же заняться вопросом наследования прав

      grant select on all tables in schema testnm to testread;


* сделайте select * from testnm.t1;

      +

* получилось?

      +

* сделайте select * from testnm.t1;

      +


* получилось?

      +

* теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);

      +


* а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?

Никто и не запрещал создавать объекты в схеме public. А раз создали таблицу, то мы становимся ее владельцем и имеем полные права на нее


* есть идеи как убрать эти права? если нет - смотрите шпаргалку

      REVOKE ALL ON SCHEMA public FROM readonly,testread;
      REVOKE ALL ON ALL TABLES IN SCHEMA public FROM testread, readonly;

* теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);

      +


* расскажите что получилось и почему

Таблица создается, но не вставляются данные в t2 т.к. прав нет, мы их забрали
