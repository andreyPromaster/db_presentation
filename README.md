# db_presentation
## Изоляция транзакций
**Транзакцией** называется множество операций, выполняемое приложением, которое переводит базу данных из одного корректного состояния в другое корректное состояние (согласованность) при условии, что транзакция выполнена полностью (атомарность) и без помех со стороны других транзакций (изоляция).
Это определение объединяет три первые буквы аббревиатуры ACID. Они настолько тесно связаны друг с другом, что рассматривать одно без другого просто нет смысла. На самом деле сложно оторвать и букву D (durability). Ведь при крахе системы в ней остаются изменения незафиксированных транзакций, с которыми приходится что-то делать, чтобы восстановить согласованность данных.
**Aномалии в стандарте SQL**
- Потерянное обновление. Такая аномалия возникает, когда две транзакции читают одну и ту же строку таблицы, затем одна транзакция обновляет эту строку, а после этого вторая транзакция тоже обновляет ту же строку, не учитывая изменений, сделанных первой транзакцией.
- Грязное чтение. Такая аномалия возникает, когда транзакция читает еще не зафиксированные изменения, сделанные другой транзакцией.
- Неповторяющееся чтение. Аномалия неповторяющегося чтения возникает, когда транзакция читает одну и ту же строку два раза, и в промежутке между чтениями вторая транзакция изменяет (или удаляет) эту строку и фиксирует изменения. Тогда первая транзакция получит разные результаты.
- Фантомное чтение. Фантомное чтение возникает, когда транзакция два раза читает набор строк по одному и тому же условию, и в промежутке между чтениями вторая транзакция добавляет строки, удовлетворяющие этому условию (и фиксирует изменения). Тогда первая транзакция получит разные наборы строк.
- Другие аномалии

Уровни изоляции в PostgreSQL
Со временем на смену блокировочным протоколам управления транзакциями пришел протокол изоляции на основе снимков (Snapshot Isolation). Его идея состоит в том, что каждая транзакция работает с согласованным снимком данных на определенный момент времени, в который попадают только те изменения, которые были зафиксированы до момента создания снимка.
Такая изоляция автоматически не допускает грязное чтение. Формально в PostgreSQL можно указать уровень Read Uncommitted, но работать она будет точно так же, как Read Committed. 

   ![image](https://user-images.githubusercontent.com/55237200/129458493-403cf2b1-a4ea-4068-97ba-e1a1512f117a.png)

  Пример создания транзакции в SQL:
~~~ 
BEGIN  ISOLATION LEVEL REPEATABLE READ;
UPDATE accounts SET balance = balance + 100.00 WHERE acctnum = 12345;
UPDATE accounts SET balance = balance - 100.00 WHERE acctnum = 7534;
COMMIT;
~~~

