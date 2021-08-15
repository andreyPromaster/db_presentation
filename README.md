# db_presentation
## Изоляция транзакций
**Транзакцией** называется множество операций, выполняемое приложением, которое переводит базу данных из одного корректного состояния в другое корректное состояние (согласованность) при условии, что транзакция выполнена полностью (атомарность) и без помех со стороны других транзакций (изоляция).
Это определение объединяет три первые буквы аббревиатуры ACID. Они настолько тесно связаны друг с другом, что рассматривать одно без другого просто нет смысла. На самом деле сложно оторвать и букву D (durability). Ведь при крахе системы в ней остаются изменения незафиксированных транзакций, с которыми приходится что-то делать, чтобы восстановить согласованность данных.
***
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
~~~sql
BEGIN  ISOLATION LEVEL REPEATABLE READ;
UPDATE accounts SET balance = balance + 100.00 WHERE acctnum = 12345;
UPDATE accounts SET balance = balance - 100.00 WHERE acctnum = 7534;
COMMIT;
~~~
***
# Индексы
Индексы — специальные объекты базы данных, предназначенные в основном для ускорения доступа к данным. Это вспомогательные структуры: любой индекс можно удалить и восстановить заново по информации в таблице. Иногда приходится слышать, что СУБД может работать и без индексов, просто медленно. Однако это не так, ведь индексы служат также для поддержки некоторых ограничений целостности.
Обычно чем больше индексов, тем больше производительность запросов к базе данных. Однако при излишнем увеличении количества индексов падает производительность операций изменения данных (вставка/изменение/удаление), увеличивается размер БД, поэтому к добавлению индексов следует относиться осторожно.
Некоторые общие принципы, связанные с созданием индексов:
- индексы необходимо создавать для столбцов, которые используются в джойнах, по которым часто производится поиск и операции сортировки. При этом необходимо учесть, что индексы всегда автоматически создаются для столбцов, на которые накладывается ограничение primary key. Чаще всего они создаются и для столбцов с foreign key (в Access - автоматически);
- индекс обязательно в автоматическом режиме создается для столбцов, на которые наложено ограничение уникальности;
- если поиск постоянно производится по определенному набору столбцов (одновременно), то в этом случае, возможно, есть смысл создать составной индекс - один индекс для группы столбцов, нужно смотреть от возможностей СУБД;
- при внесении изменений в таблицы автоматически изменяются и индексы, наложенные на эту таблицу. В результате индекс может быть сильно фрагментирован, что сказывается на производительности.

## Типы индексов
PostgreSQL поддерживает несколько типов индексов: B-дерево, хеш, GiST, SP-GiST, GIN(полнотекстового поиска) и BRIN(Block Range Index). Для разных типов индексов применяются разные алгоритмы, ориентированные на определённые типы запросов. По умолчанию команда CREATE INDEX создаёт индексы типа B-дерево, эффективные в большинстве случаев.
- Индекс btree, он же B-дерево, пригоден для данных, которые можно отсортировать. Иными словами, для типа данных должны быть определены операторы «больше», «больше или равно», «меньше», «меньше или равно» и «равно». Вот схематичный пример индекса по одному полю с целочисленными ключами.
Рассмотрим поиск значения в дереве по условию «индексированное-поле = выражение».
![image](https://user-images.githubusercontent.com/55237200/129477010-e843d58b-313e-4ff7-8d4c-73872bfab3b1.png)
- Хеш-индексы работают только с простыми условиями равенства. Планировщик запросов может применить хеш-индекс, только если индексируемый столбец участвует в сравнении с оператором =. Создать такой индекс можно следующей командой:
~~~sql
CREATE INDEX имя ON таблица USING HASH (столбец);
~~~
Идея хеширования состоит в том, чтобы значению любого типа данных сопоставить некоторое небольшое число (от 0 до N−1, всего N значений). Такое сопоставление называют хеш-функцией. Полученное число можно использовать как индекс обычного массива, куда и складывать ссылки на строки таблицы (TID). Элементы такого массива называют корзинами хеш-таблицы в одной корзине могут лежать несколько TID-ов, если одно и то же проиндексированное значение встречается в разных строках.
Хеш-функция тем лучше, чем равномернее она распределяет исходные значения по корзинам. Но даже хорошая функция будет иногда давать одинаковый результат для разных входных значений — это называется коллизией.
- GiST-индексы представляют собой не просто разновидность индексов, а инфраструктуру, позволяющую реализовать много разных стратегий индексирования. Как следствие, GiST-индексы могут применяться с разными операторами, в зависимости от стратегии индексирования (класса операторов). Например, стандартный дистрибутив PostgreSQL включает классы операторов GiST для нескольких двумерных типов геометрических данных, что позволяет применять индексы в запросах с операторами:
~~~
<<
&<
&>
>>
<<|
~~~
Индексы можно создавать и по нескольким столбцам таблицы. Например, если у вас есть таблица:
~~~sql
CREATE TABLE test2 (
  major int,
  minor int,
  name varchar
);
~~~
вы часто выполняете запросы вида:
~~~sql
SELECT name FROM test2 WHERE major = константа AND minor = константа;
~~~
тогда имеет смысл определить индекс, покрывающий оба столбца major и minor. Например:
~~~sql
CREATE INDEX test2_mm_idx ON test2 (major, minor);
~~~
В настоящее время составными могут быть только индексы типов B-дерево, GiST, GIN и BRIN. Число столбцов в индексе ограничивается 32.

***
## Полезные ссылки
[Как устроены базы данных](https://habr.com/ru/company/oleg-bunin/blog/358984/)
[Что такое изоляция и почему это важно](https://habr.com/ru/company/postgrespro/blog/442804/)


