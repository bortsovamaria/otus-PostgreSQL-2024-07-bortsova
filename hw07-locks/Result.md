

* Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
    Меняем параметры СУБД Postgres
    * Включаем логирование блокировок логах Postgres
      
            alter system set log_lock_waits=on; 
      
    * Включаем логирование блокировок, которые удерживаются более 200 миллисекунд
      
          alter system set deadlock_timeout=200;
      
    * Применяем параметры к Postgres.
      
          select pg_reload_conf();
     
    * Данные из лога Postgres, где видно срабатывание. Данные от задания ниже.
     
           2024-09-17 20:04:22.428 UTC [4493] postgres@locks LOG:  process 4493 still waiting for ShareLock on transaction 31723 after 200.512 ms
           2024-09-17 20:04:22.428 UTC [4493] postgres@locks DETAIL:  Process holding the lock: 4490. Wait queue: 4493.
           2024-09-17 20:04:22.428 UTC [4493] postgres@locks CONTEXT:  while updating tuple (0,10) in relation "accounts"
           2024-09-17 20:04:22.428 UTC [4493] postgres@locks STATEMENT:  UPDATE accounts SET amount = amount + 380.00 WHERE acc_no = 1; 
          
    

* Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.
    Session 1
    
        BEGIN;
        SELECT txid_current(), pg_backend_pid(); --Session_1: 31725 : 4490
        UPDATE accounts 
        SET amount = amount + 100.00 WHERE acc_no = 1; 
       
    Session 2
    
        BEGIN;
        SELECT txid_current(), pg_backend_pid(); --Session_2: 31726 : 4492
        UPDATE accounts SET amount = amount - 50.00 WHERE acc_no = 1;
    
    Session 3

        BEGIN;
        SELECT txid_current(), pg_backend_pid(); --Session_3: 31727 : 4493
        UPDATE accounts SET amount = amount + 3800.00 WHERE acc_no = 1;

    Session 1 all locks
    
        SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
          FROM pg_locks 
          WHERE pid = 4490;
        
           locktype    |   relation    | virtxid |  xid  |       mode       | granted 
        ---------------+---------------+---------+-------+------------------+---------
         relation      | pg_locks      |         |       | AccessShareLock  | t
         relation      | accounts_pkey |         |       | RowExclusiveLock | t
         relation      | accounts      |         |       | RowExclusiveLock | t
         virtualxid    |               | 7/391   |       | ExclusiveLock    | t
         transactionid |               |         | 31725 | ExclusiveLock    | t
        (5 rows)

    * Блокировки первой сессии с pid = 4490
        * AccessShareLock на таблице pg_locks и уже полученная блокировка granted=t. Обычная блокировка при SELECT из таблицы.
        * RowExclusiveLock на таблице accounts и ее первичном ключе accounts_pkey. На первичном ключе (primary key) всегда есть индекс. Эта блокировка операции UPDATE.
        * ExclusiveLock на виртуальном идентификаторе транзакции (virtualxid), которая реально не меняет данные (virtxid=7/391). В нашем случае она есть т.к. мы начали транзакцию через BEGIN
        * ExclusiveLock исключительная блокировка настоящего номера транзакции (transactionid), которая начала менять данные (xid=31725).
      
               
          SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
            FROM pg_locks 
            WHERE pid = 4492;
            
             locktype    |   relation    | virtxid |  xid  |       mode       | granted 
          ---------------+---------------+---------+-------+------------------+---------
           relation      | accounts_pkey |         |       | RowExclusiveLock | t
           relation      | accounts      |         |       | RowExclusiveLock | t
           virtualxid    |               | 6/24    |       | ExclusiveLock    | t
           transactionid |               |         | 31725 | ShareLock        | f
           tuple         | accounts      |         |       | ExclusiveLock    | t
           transactionid |               |         | 31726 | ExclusiveLock    | t
          (6 rows)
          

* Блокировки второй сессии с pid = 4492
        * RowExclusiveLock на таблице accounts и ее первичном ключе accounts_pkey. На первичном ключе (primary key) всегда есть индекс. Эта блокировка операции UPDATE.
        * ExclusiveLock на виртуальном идентификаторе транзакции (virtualxid), которая реально не меняет данные (virtxid=6/24). В нашем случае она есть т.к. мы начали транзакцию через BEGIN
        * ShareLock. Транзакция пытается наложить блокировку ShareLock на xid=31725, но не может granted=f  
        * ExclusiveLock (tuple) возникает, когда несколько сессий пытаются изменить одни и те же данные. Чтобы не ссылаться на одни и те же идентификаторы транзакций, формируется некий общий tuple с блокровкой на таблицу.
        * ExclusiveLock исключительная блокировка настоящего номера транзакции (transactionid), которая начала менять данные (xid=31726).

      
      SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
        FROM pg_locks 
        WHERE pid = 4493;
        
         locktype    |   relation    | virtxid |  xid  |       mode       | granted 
      ---------------+---------------+---------+-------+------------------+---------
       relation      | accounts_pkey |         |       | RowExclusiveLock | t
       relation      | accounts      |         |       | RowExclusiveLock | t
       virtualxid    |               | 5/11    |       | ExclusiveLock    | t
       tuple         | accounts      |         |       | ExclusiveLock    | f
       transactionid |               |         | 31727 | ExclusiveLock    | t
      (5 rows)
         
* Блокировки третьей сессии с pid = 4493
        * RowExclusiveLock на таблице accounts и ее первичном ключе accounts_pkey. На первичном ключе (primary key) всегда есть индекс. Эта блокировка операции UPDATE.
        * ExclusiveLock на виртуальном идентификаторе транзакции (virtualxid), которая реально не меняет данные (virtxid=5/11). В нашем случае она есть т.к. мы начали транзакцию через BEGIN
        * ExclusiveLock (tuple) возникает, когда несколько сессий пытаются изменить одни и те же данные. Чтобы не ссылаться на одни и те же идентификаторы транзакций, формируется некий общий tuple с блокровкой на таблицу. В данном случае не получилось наложить блокировку на таблицу (granted=f) 
        * ExclusiveLock исключительная блокировка настоящего номера транзакции (transactionid), которая начала менять данные (xid=31727).


* Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

        --Session 1, transaction 1
        BEGIN;
        UPDATE accounts 
        SET amount = amount + 100.00 WHERE acc_no = 1;

        --Session 2, transaction 1
        BEGIN;
        UPDATE accounts 
        SET amount = amount + 200.00 WHERE acc_no = 2;

        --Session 3, transaction 1
        BEGIN;
        UPDATE accounts 
        SET amount = amount + 300.00 WHERE acc_no = 3;

        --Session 1, transaction 2
        BEGIN;
        UPDATE accounts 
        SET amount = amount + 100.00 WHERE acc_no = 3;
    
        --Session 2, transaction 2
        BEGIN;
        UPDATE accounts 
        SET amount = amount + 200.00 WHERE acc_no = 1;
    
        --Session 3, transaction 2
        BEGIN;
        UPDATE accounts 
        SET amount = amount + 300.00 WHERE acc_no = 2;

  
* Лог Postgres

        2024-09-17 21:32:38.198 UTC [4493] postgres@locks LOG:  process 4493 detected deadlock while waiting for ShareLock on transaction 31729 after 200.487 ms
        2024-09-17 21:32:38.198 UTC [4493] postgres@locks DETAIL:  Process holding the lock: 4492. Wait queue: .
        2024-09-17 21:32:38.198 UTC [4493] postgres@locks CONTEXT:  while updating tuple (0,2) in relation "accounts"
        2024-09-17 21:32:38.198 UTC [4493] postgres@locks STATEMENT:  UPDATE accounts
                SET amount = amount + 300.00 WHERE acc_no = 2;
        2024-09-17 21:32:38.198 UTC [4493] postgres@locks ERROR:  deadlock detected
        2024-09-17 21:32:38.198 UTC [4493] postgres@locks DETAIL:  Process 4493 waits for ShareLock on transaction 31729; blocked by process 4492.
                Process 4492 waits for ShareLock on transaction 31728; blocked by process 4490.
                Process 4490 waits for ShareLock on transaction 31730; blocked by process 4493.
                Process 4493: UPDATE accounts
                SET amount = amount + 300.00 WHERE acc_no = 2;
                Process 4492: UPDATE accounts
                SET amount = amount + 200.00 WHERE acc_no = 1;
                Process 4490: UPDATE accounts
                SET amount = amount + 100.00 WHERE acc_no = 3;
        2024-09-17 21:32:38.198 UTC [4493] postgres@locks HINT:  See server log for query details.
        2024-09-17 21:32:38.198 UTC [4493] postgres@locks CONTEXT:  while updating tuple (0,2) in relation "accounts"
        2024-09-17 21:32:38.198 UTC [4493] postgres@locks STATEMENT:  UPDATE accounts
                SET amount = amount + 300.00 WHERE acc_no = 2;
        2024-09-17 21:32:38.200 UTC [4490] postgres@locks LOG:  process 4490 acquired ShareLock on transaction 31730 after 21450.657 ms
        2024-09-17 21:32:38.200 UTC [4490] postgres@locks CONTEXT:  while updating tuple (0,3) in relation "accounts"


* Результат:
        * В первой сессии прошли оба запроса.
        * Во второй сессии второй запрос висит в ожидании сняти блокировки.
        * В третьей сессии транзакция прервалась с логом
          
          ERROR:  deadlock detected
          DETAIL:  Process 4493 waits for ShareLock on transaction 31729; blocked by process 4492.
          Process 4492 waits for ShareLock on transaction 31728; blocked by process 4490.
          Process 4490 waits for ShareLock on transaction 31730; blocked by process 4493.
          HINT:  See server log for query details.
          CONTEXT:  while updating tuple (0,2) in relation "accounts"

    * Вывод. По логам postgres можно разобраться с блокировками, но это совсем не просто. Информация о блокировка у нас попадает т.к. мы настроили ее в самом начале.


* Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
    Взаимоблокировка 2-х транзакций, выполняющих UPDATE одной и той же таблицы (без where) возможна, если одна команда будет обновлять строки таблицы в прямом порядке, а другая - в обратном. Такое может произойти в реальной жизни (но это маловероятно), если для команд будут построены разные планы выполнения, например, одна будет читать таблицу последовательно, а другая - по индексу

