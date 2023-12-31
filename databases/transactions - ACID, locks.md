## ACID  
*	**Atomicity** (атомарность) — выражается в том, что транзакция должна быть выполнена в целом или не выполнена вовсе.  
*	**Consistency** (согласованность) — гарантирует, что по мере выполнения транзакций, данные переходят из одного согласованного состояния в другое, то есть транзакция не может разрушить взаимной согласованности данных.  
*	**Isolation** (изолированность) — локализация пользовательских процессов означает, что конкурирующие за доступ к БД транзакции физически обрабатываются последовательно, изолированно друг от друга, но для пользователей это выглядит, как будто они выполняются параллельно.  
*	**Durability** (долговечность) — устойчивость к ошибкам — если транзакция завершена успешно, то те изменения в данных, которые были ею произведены, не могут быть потеряны ни при каких обстоятельствах.  
  
  
## Аномалии
1.	потерянное обновление (**lost update**) — при одновременном изменении одного блока данных разными транзакциями теряются все изменения, кроме последнего;
2.	«грязное» чтение (**dirty read**) — чтение данных, добавленных или изменённых транзакцией, которая впоследствии не подтвердится (откатится);
3.	неповторяющееся чтение (non-repeatable read) — при повторном чтении в рамках одной транзакции ранее прочитанные данные оказываются изменёнными;
4.	фантомное чтение (**phantom reads**) — одна транзакция в ходе своего выполнения несколько раз выбирает множество строк по одним и тем же критериям. Другая транзакция в интервалах между этими выборками добавляет строки или изменяет столбцы некоторых строк, используемых в критериях выборки первой транзакции, и успешно заканчивается. В результате получится, что одни и те же выборки в первой транзакции дают разные множества строк.
5.	**Read skew** - фактически, это non-repeatable read, но в составе нескольких таблиц.
6.	**Write skew** - Транзакция что-то считывает, принимает решение на основе увиденного значения и записывает решение в базу данных. Однако к моменту записи предпосылка решения уже не соответствует действительности. Только serializable предотвращает эту аномалию

| **Уровень изоляции** | **other**  | **phantom reads** | **non-repeatable read** | **dirty read** | **lost update** |
| -------------------- | ---------- | ----------------- | ----------------------- | -------------- | --------------- |
| SERIALIZABLE         | +          | +                 | +                       | +              | +               |
| REPEATABLE READ      | могут быть | \-                | +                       | +              | +               |
| READ COMMITTED       | могут быть | \-                | \-                      | +              | +               |
| READ UNCOMMITTED     | могут быть | \-                | \-                      | \-             | +               |


## Подходы к реализации уровней изоляции
Существует два различных подхода к реализации изолированности: 
1. Версионирование (**snapshot**) означает, что транзакции будут работать со своей копией данных, не влияя друг на друга, но впоследствии несколько изменённых копий надо будет как-то слить в одну. Строгость версионирования регулирует моменты (когда) и размеры (сколько данных копировать) этих копий.
2. Блокирование (**lock**) означает, что одна транзакция будет ждать другую, чтобы избежать побочных эффектов, в зависимости от строгости уровней изоляции этих транзакций. Различные виды блокировок обеспечивают более гранулярное блокирование, например, только по строкам, или только на небольшой кусочек транзакции, а не на всю.

В описаниях блокировок часто всплывает понятие оптимистичной и пессимистичной блокировки. Это два высокоуровневых термина, которые обозначают следующее:

Пессимистичная блокировка происходит как можно раньше, чтобы предотвратить как можно больше побочных эффектов, но транзакции чаще ждут друг-друга. Например, блокирование на обновление строк, которые прочитала транзакция в режиме read committed — это пессимистичная блокировка, потому что она происходит, как только ваша транзакция прочитала строки, не смотря на то, будет ли их кто-то обновлять или нет.

Оптимистичная блокировка происходит как можно позже, чтобы транзакции реже ждали друг-друга. Плата за такой вид блокировки — необходимость обрабатывать “конфликт обновления”. Конфликт обновления происходит, например, когда “ваша” транзакция прочитала строки и хочет их обновить, но другая транзакция их уже обновила — при оптимистичной блокировке эта ошибка возникнет в “вашей” транзакции и её надо корректно обработать: как минимум, повторить транзакцию позже, как максимум, обработать ошибку прямо в коде транзакции.

## Блокировки
*	Разделяемая блокировка (**shared lock**) - Резервирует ресурс (страницу или строку) только для чтения. Другие процессы не могут изменять заблокированный таким образом ресурс, но, с другой стороны, несколько процессов могут одновременно накладывать разделяемую блокировку на один и тот же ресурс. Иными словами, чтение ресурса с разделяемой блокировкой могут одновременно выполнять несколько процессов.
*	Монопольная блокировка (**exclusive lock**) - Резервирует страницу или строку для монопольного использования одной транзакции. Блокировка этого типа применяется инструкциями DML (INSERT, UPDATE и DELETE), которые модифицируют ресурс. Монопольную блокировку нельзя установить, если на ресурс уже установлена разделяемая или монопольная блокировка другим процессом, т.е. на ресурс может быть установлена только одна монопольная блокировка. На ресурс (страницу или строку) с установленной монопольной блокировкой нельзя установить никакую другую блокировку.
*	Блокировка обновления (**update lock**) - Может быть установлена на ресурс только при отсутствии на нем другой блокировки обновления или монопольной блокировки. С другой стороны, этот тип блокировки можно устанавливать на объекты с установленной разделяемой блокировкой. В таком случае блокировка обновления накладывает на объект другую разделяемую блокировку. Если транзакция, которая модифицирует объект, подтверждается, и у объекта нет никаких других блокировок, блокировка обновления преобразовывается в монопольную блокировку. У объекта может быть только одна блокировка обновления.
*	Взаимоблокировка (**deadlock**) — это особая проблема одновременного конкурентного доступа, в которой две транзакции блокируют друг друга. В частности, первая транзакция блокирует объект базы данных, доступ к которому хочет получить другая транзакция, и наоборот. Взаимоблокировка может быть вызвана несколькими транзакциями, которые создают цикл зависимостей.


 
---
[Уровень изолированности транзакций — Википедия](https://ru.m.wikipedia.org/wiki/%D0%A3%D1%80%D0%BE%D0%B2%D0%B5%D0%BD%D1%8C_%D0%B8%D0%B7%D0%BE%D0%BB%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%BD%D0%BE%D1%81%D1%82%D0%B8_%D1%82%D1%80%D0%B0%D0%BD%D0%B7%D0%B0%D0%BA%D1%86%D0%B8%D0%B9)  
[Уровни изоляции транзакций с примерами на PostgreSQL / Хабр](https://habr.com/ru/articles/317884/)  
[Уровни изолированности транзакций для самых маленьких / Хабр](https://habr.com/ru/articles/469415/)  
[Уровни изоляции транзакций в Postgresql](https://gadjimuradov.ru/post/urovni-izolyacii-tranzakcij-v-postgresql/)  
[К чему может привести ослабление уровня изоляции транзакций в базах данных / Habr (тут про skew)](https://habr.com/ru/amp/publications/501294/)  
[Слабые уровни изоляции транзакций на примерах - YouTube](https://www.youtube.com/watch?v=HVGUGNC5iOk&list=WL&index=15&ab_channel=BPMN2ru)  
[Проблемы при ослаблении изоляции транзакций. Read Skew, Write Skew - YouTube](https://www.youtube.com/watch?v=brKxnP_YY30&ab_channel=SergeyNemchinskiy)  
[Transact-SQL Блокировки](https://professorweb.ru/my/sql-server/2012/level3/3_15.php)  
[Транзакции - Подготовка к собеседованию](https://folko.gitbook.io/podgotovka-k-sobesedovaniyu/bd/tranzakcii)
