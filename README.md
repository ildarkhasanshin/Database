Rus
===

Что такое Database?
---

Database — библиотека классов на PHP 5.3 для простой, удобной, быстрой и безопасной работы с базой данных MySql, использующая расширение PHP `mysqli`.


Зачем в 2017 году нужен самописный класс для MySql, если в PHP есть абстракция PDO и расширение mysqli?
---

Основные недостатки всех библиотек для работы с базой в PHP это:

* **Многословность**
  * Что бы предотвратить <a href="http://ru.wikipedia.org/wiki/%D0%92%D0%BD%D0%B5%D0%B4%D1%80%D0%B5%D0%BD%D0%B8%D0%B5_SQL-%D0%BA%D0%BE%D0%B4%D0%B0">SQL-инъекции</a>, у разработчиков есть два пути:
    * Использовать <a href="http://php.net/manual/ru/mysqli.quickstart.prepared-statements.php">подготавливаемые запросы</a> (prepared statements).
    * Вручную экранировать параметры идущие в тело SQL-запроса. Строковые параметры прогонять через <a href="http://php.net/manual/ru/mysqli.real-escape-string.php">mysqli_real_escape_string()</a>, а ожидаемые числовые параметры приводить к соответствующим типам — `int` и `float`.
  * Оба подхода имеют колоссальные недостатки:
    * Подготавливаемые запросы <a target="_blank" rel="nofollow" href="http://php.net/manual/ru/mysqli.prepare.php#refsect1-mysqli.prepare-examples">ужасно многословны</a>. Пользоваться "из коробки" абстракцией PDO или расширением mysqli, без агрегирования всех методов для получения данных из СУБД просто невозможно — что бы получить значение из таблицы необходимо написать минимум 5 строк кода! И так на каждый запрос!
    * Экранирование вручную параметров, идущих в тело SQL-запроса — даже не обсуждается. Хороший программист — ленивый программист. Всё должно быть максимально автоматизировано.
* **Невозможность получить SQL запрос для отладки**
  * Что бы понять, почему в программе не работает SQL-запрос, его нужно отладить — найти либо логическую, либо синтаксическую ошибку. Что бы найти ошибку, необходимо "видеть" сам SQL-запрос, на который "ругнулась" база, с подставленными в его тело параметрами. Т.е. иметь сформированный полноценный SQL.
Если разработчик использует PDO, с подготавливаемыми запросами, то это сделать... НЕВОЗМОЖНО! Никаких максимально удобных механизмов для этого в родных библиотеках НЕ ПРЕДУСМОТРЕНО. Остается либо извращаться, либо лезть в лог базы данных.


Решение: Database — класс для работы с MySql
---
1. Избавляет от многословности — вместо 3 и более строк кода для исполнения одного запроса при использовании "родной" библиотеки, вы пишите всего 1!
2. Экранирует все параметры, идущие в тело запроса, согласно указанному типу заполнителей — надежная защита от SQL-инъекций.
3. Не замещает функциональность "родного" mysqli адаптера, а просто дополняет его.


Что такое placeholders (заполнители)?
---

Placeholders (англ. — заполнители) — специальные типизированные маркеры, которые пишутся в строке SQL запроса вместо явных значений (параметров запроса). А сами значения передаются "позже", в качестве последующих аргументов основного метода, выполняющего SQL-запрос:

```php
<?php
// Соединение с СУБД и получение объекта Database_Mysql
// Database_Mysql - "обертка" над "родным" объектом mysqli
$db = Database_Mysql::create("localhost", "root", "password")
      // Выбор базы данных
      ->setDatabaseName("test")
      // Выбор кодировки
      ->setCharset("utf8");

// Получение объекта результата Database_Mysql_Statement
// Database_Mysql_Statement - "обертка" над "родным" объектом mysqli_result
$result = $db->query("SELECT * FROM `users` WHERE `name` = '?s' AND `age` = ?i", "Василий", 30);

// Получаем данные (в виде ассоциативного массива, например)
$data = $result->fetch_assoc();

// Не работает запрос? Не проблема - выведите его на печать:
echo $db->getQueryString();
```

Параметры SQL-запроса, прошедшие через систему placeholders, обрабатываются специальными функциями экранирования, в зависимости от типа заполнителей. Т.е. вам теперь нет необходимости заключать переменные в функции экранирования типа `mysqli_real_escape_string()` или приводить их к числовому типу, как это было раньше:

```php
<?php
// Раньше перед каждым запросом в СУБД мы делали
// примерно это (а многие и до сих пор `это` не делают):
$id = (int) $_POST['id'];
$value = mysql_real_escape_string($_POST['value'], $link);
$result = mysql_query("SELECT * FROM `t` WHERE `f1` = '$value' AND `f2` = $id", $link);
```

Теперь запросы стало писать легко, быстро, а главное библиотека Database полностью предотвращает любые возможные SQL-инъекции.


Типы заполнителей и типы параметров SQL-запроса
---

Типы заполнителей и их предназначение описываются ниже. Прежде чем знакомиться с типами заполнителей, необходимо понять как работает механизм библиотеки Database.

```php
 $db->query("SELECT ?i", 123); 
```
SQL-запрос после преобразования шаблона:
```sql
SELECT 123
```
В процессе исполнения этой команды библиотека проверяет, является ли аргумент `123` целочисленным значением. Заполнитель `?i` представляет собой символ `?` (знак вопроса) и первую букву слова `integer`. Если аргумент действительно представляет собой целочисленный тип данных, то в шаблоне SQL-запроса заполнитель `?i` заменяется на значение `123` и SQL передается на исполнение.

Поскольку PHP слаботипизированный язык, то вышеописанное выражение эквивалентно нижеописанному:

```php
 $db->query("SELECT ?i", '123'); 
 ```
 
 SQL-запрос после преобразования шаблона:
 
 ```sql
 SELECT 123
 ```
 
 т.е. числа (целые и с плавающей точкой) представленные как в своем типе, так и в виде `string` — равнозначны с точки зрения библиотеки.
 
 ### Приведение к типу заполнителя
 
 ```php
  $db->query("SELECT ?i", '123.7'); 
  ```
  SQL-запрос после преобразования шаблона:
  ```sql
  SELECT 123
  ```
  
  В данном примере заполнитель целочисленного типа данных ожидает значение типа `integer`, а передается `double`. **По-умолчанию библиотека работает в режиме приведения типов, что дало в итоге приведение типа `double` к `int`**.
  
  Режимы работы библиотеки и принудительное приведение типов
  ----
  Существует два режима работы библиотеки:
  
  * **Database_Mysql::MODE_STRICT** — строгий режим соответствия типа заполнителя и типа аргумента.
    В режиме MODE_STRICT аргументы должны соответствовать типу заполнителя. Например, попытка передать в качестве аргумента значение `55.5` или `'55.5'` для заполнителя целочисленного типа `?i` приведет к выбросу исключения:
  
```php
  // устанавливаем строгий режим работы
$db->setTypeMode(Database_Mysql::MODE_STRICT);
// это выражение не будет исполнено, будет выброшено исключение:
// Попытка указать для заполнителя типа int значение типа double в шаблоне запроса SELECT ?i
$db->query('SELECT ?i', 55.5);
```

* **Database_Mysql::MODE_TRANSFORM** — режим преобразования аргумента к типу заполнителя при несовпадении типа заполнителя и типа аргумента. Режим MODE_TRANSFORM является "толерантным" режимом и при несоответствии типа заполнителя и типа аргумента не генерирует исключение, а **пытается преобразовать аргумент к нужному типу заполнителя посредством самого языка PHP**.

Допускаются следующие преобразования:

* К типу `int` (заполнитель `?i`) приводятся
  * числа с плавающей точкой, представленные как `string` или тип `double`
  * `bool` TRUE преобразуется в `int(1)`, FALSE преобразуется в `int(0)`
  * null преобразуется в `int(0)`
* К типу `double` (заполнитель `?d`) приводятся
  * целые числа, представленные как строка или тип `int`
  * `bool` TRUE преобразуется в `float(1)`, FALSE преобразуется в `float(0)`
  * `null` преобразуется в `float(0)`
* К типу `string` (заполнитель `?s`) приводятся
  * `bool` TRUE преобразуется в `string(1) "1"`, FALSE преобразуется в `string(1) "0"`.
  ```
  Это поведение отличается от приведения типа bool к int в PHP, т.к. зачастую булев тип записывается в MySql именно как число.
  Почему в PHP приведение bool FALSE к string даёт string(1) "", а bool TRUE — string(1) "1" — не понятно.
  ```
  * значение типа `numeric` преобразуется в строку согласно правилам преобразования PHP
  * `NULL` преобразуется в `string(0) ""`
* К типу `null` (заполнитель `?n`) приводятся
  * любые аргументы
  
Для массивов, объектов и ресурсов преобразования не допускаются.
  
Какие типы заполнителей представлены в библиотеке Database?
---

### `?i` — заполнитель целого числа

В режиме **MODE_TRANSFORM** данные типов `double`, `boolean`, `NULL` принудительно приводятся к типу integer согласно правилам преобразования к типу integer в PHP.

```php
 $_POST['id'] = '123456';
$db->query('SELECT * FROM `users` WHERE `id` = ?i', $_POST['id']); 
```
SQL-запрос после преобразования шаблона:
```sql
SELECT * FROM `users` WHERE `id` = 123456
```
**ВНИМАНИЕ!** Если вы оперируете числами, выходящими за пределы PHP_INT_MAX, то:
* Оперируйте ими исключительно как строками в своих программах.
* Не используйте данный заполнитель, используйте заполнитель строки `?s` (см. ниже). Дело в том, что числа, выходящие за пределы PHP_INT_MAX, PHP интерпретирует как числа с плавающей точкой. Парсер библиотеки постарается преобразовать параметр к типу int, в итоге «*результат будет неопределенным, так как float не имеет достаточной точности, чтобы вернуть верный результат. В этом случае не будет выведено ни предупреждения, ни даже замечания!*» — <a href="http://php.net/manual/ru/language.types.integer.php#language.types.integer.casting.from-float">php.net</a>. 

### `?d` — заполнитель числа с плавающей точкой

В режиме **MODE_TRANSFORM** данные типов (`integer`, `boolean`, `NULL`) принудительно приводятся к типу `double` согласно правилам преобразования к типу `double` в PHP.

**ВНИМАНИЕ!** Если вы используете библиотеку для работы с типом данных `double`, установите соответствующую локаль, что бы разделитель целой и дробной части был одинаков как на уровне PHP, так и на уровне СУБД. 

### `?s` — заполнитель строкового типа

В режиме **MODE_TRANSFORM** скалярные данные типов (`integer`, `double`, `NULL`) принудительно приводятся к типу `string` согласно правилам преобразования к типу `string` в PHP. `boolean` преобразуется в 1 или 0. Далее значения экранируются с помощью функции PHP `mysqli_real_escape_string()`.

```php
 $db->query('SELECT "?s", "?s", "?s", "?s", "?s"', 55.5, true, false, null, 'Д"Артаньян'); 
 ```
 SQL-запрос после преобразования шаблона:
 
```sql
SELECT "55.5", "1", "0", "", "Д\"Артаньян"
```
### `?S` — заполнитель строкового типа для подстановки в SQL-оператор LIKE 

В режиме **MODE_TRANSFORM** скалярные данные типов `integer`, `double`, `NULL` принудительно приводятся к типу `string` согласно правилам преобразования к типу `string` в PHP. `boolean` преобразуется в 1 или 0. Далее значения экранируются с помощью функции PHP `mysqli_real_escape_string()` + экранирование спецсимволов, используемых в операторе LIKE (`%` и `_`). 

```php
 $db->query('SELECT "?S"', '% _'); 
 ```
 SQL-запрос после преобразования шаблона:
 ```sql
 SELECT "\% \_"
 ```
 
 ### `?n` — заполнитель `NULL` типа
В режиме **MODE_TRANSFORM** любые параметры запроса игнорируются, заполнители заменяются на строку `NULL` в SQL запросе:

```php
 $db->query('SELECT ?n', 123); 
 ```
 SQL-запрос после преобразования шаблона:
 
 ```sql
 SELECT NULL
 ```
 
Ограничивающие кавычки
---

Библиотека **требует** от программиста соблюдения синтаксиса SQL. Это значит, что следующий запрос работать не будет:

```php
$db->query('SELECT CONCAT("Hello, ", ?s, "!")', 'world');
```

— заполнитель `?s` необходимо взять в одинарные или двойные кавычки:

```php
$db->query('SELECT concat("Hello, ", "?s", "!")', 'world');
```

SQL-запрос после преобразования шаблона:

```sql
SELECT concat("Hello, ", "world", "!")
```

Для тех, кто привык работать с PDO это покажется странным, но реализовать механизм, определяющий, нужно ли в одном случае заключать значение заполнителя в кавычки или нет — очень нетривиальная задача, трубующая написания целого парсера.


Чем НЕ является библиотека Database?
---

Большинство оберток под различные драйверы баз данных являются нагромождением бесполезного кода. Их авторы, сами не понимая практической цели своих оберток, превращают их в подобие построителей запросов (sql builder), ActiveRecord библиотек и прочих ORM-решений.

Библиотека Database не является ничем из перечисленных. Это лишь удобный инструмент для работы с обычным SQL в рамках СУБД MySQL — и не более!
