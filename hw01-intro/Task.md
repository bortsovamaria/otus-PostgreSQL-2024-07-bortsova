# Домашнее задание
Работа с уровнями изоляции транзакции в PostgreSQL

## Цель:
* научиться работать в Яндекс Облаке
* научиться управлять уровнем изолции транзации в PostgreSQL и понимать особенность работы уровней read commited и repeatable read

### Описание/Пошаговая инструкция выполнения домашнего задания:
1. поставить PostgreSQL
2. зайти вторым ssh (вторая сессия)
3. запустить везде psql из под пользователя postgres 
4. выключить auto commit

5. сделать  в первой сессии новую таблицу и наполнить ее данными create table persons(id serial, first_name text, second_name text); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov'); 
commit;
6. посмотреть текущий уровень изоляции: show transaction isolation level

7. начать новую транзакцию в обеих сессиях с дефолтным (не меняя) уровнем изоляции
8. в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');
9. сделать select from persons во второй сессии
10. видите ли вы новую запись и если да то почему?

11. завершить первую транзакцию - commit;
12. сделать select from persons во второй сессии
13. видите ли вы новую запись и если да то почему?
14. завершите транзакцию во второй сессии

15. начать новые но уже repeatable read транзации - set transaction isolation level repeatable read;
16. в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');
17. сделать select* from persons во второй сессии*
18. видите ли вы новую запись и если да то почему?
19. завершить первую транзакцию - commit;

20. сделать select from persons во второй сессии
21. видите ли вы новую запись и если да то почему?
22. завершить вторую транзакцию
23. сделать select * from persons во второй сессии
24. видите ли вы новую запись и если да то почему?