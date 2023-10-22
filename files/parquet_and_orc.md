## Parquet
Parquet — это бинарный, колоночно-ориентированный формат хранения больших данных, изначально созданный для экосистемы Hadoop, позволяющий использовать преимущества сжатого и эффективного колоночно-ориентированного представления информации. Паркет позволяет задавать схемы сжатия на уровне столбцов и добавлять новые кодировки по мере их появления.  
  
Parquet использует архитектуру, основанную на “уровнях определения” (definition levels) и “уровнях повторения” (repetition levels), что позволяет довольно эффективно кодировать данные, а информация о схеме выносится в отдельные метаданные.
При этом оптимально хранятся и пустые значения.
  
<img src="https://github.com/oldos-orwell/data_notes/blob/main/images/parquet.png"/>  
  
* File Header - содержит магическое число PAR1, которое позволяет прочитать и определить, что это Parquet-файл для его дальнейшей оптимизации.
* Row-group - группы строк - разбиение одного конкретного файла на множество маленьких блоков для того, чтобы в случае необходимости не приходилось вычитывать весь огромный файл. Позволяет параллельно работать с данными на уровне Map-Reduce
* Column Chunk - разбиение по колонкам - каждую колонку можно прочитать отдельно. Оптимизирует работу с диском. Если представить данные как таблицу, то они записываются не построчно, а по колонкам.  
  
| A  | C  | C  |  
|----|----|----|  
| a1 | b1 | c1 |  
| a2 | b2 | c2 |  
| a3 | b3 | c3 |

Строковое хранение: [a1, b1, c1, a2, b2, c2, a3, b3, c3]  
Колоночное хранение: [a1, a2, a3, b1, b2, b3. c1, c2, c3]
 
* Page - физическое представление данных, набор значений для каждого столбца в файле. Page имеет собственное описание схемы данных, которое указывает на тип данных, размер конкретной страницы и способ кодирования. Также как в Row-group, там возможны ссылки на словарь для того, чтобы декодировать данные. Позволяет распределять работу по кодированию и сжатию - за счёт разделения достигается довольно эффективное и быстрое кодирование, на данном уровне производится сжатие (если оно настроено).
* File Footer - содержит:  
  * Метаданные (определение схемы, информация о группах строк, метадано стоные лбцах);
  * Дополнительные данные, например, местоположение словаря для столбца;
  * Статистику по столбцам.
  
Parquet явно отделяет метаданные от данных, что позволяет разбивать столбцы на несколько файлов, а также иметь один файл метаданных, ссылающийся на несколько файлов паркета. Метаданные записываются после значащих данных, чтобы обеспечить однопроходную запись. Таким образом, сначала прочитаются метаданные файла, чтобы найти все нужные фрагменты столбцов, которые дальше будут прочтены последовательно.  
   
Достоинства хранения данных в Parquet:
* Несмотря на то, что они и созданы для hdfs, данные могут храниться и в других файловых системах, таких как GlusterFs или поверх NFS
* По сути это просто файлы, а значит с ними легко работать, перемещать, бэкапить и реплицировать.
* Колончатый вид позволяет значительно ускорить работу аналитика, если ему не нужны все колонки сразу.
* Нативная поддержка в Spark из коробки обеспечивает возможность просто взять и сохранить файл в любимое хранилище.
* Эффективное хранение с точки зрения занимаемого места.

## ORC
ORC (Optimized Row Columnar) - колоночно-ориентированный (столбцовый) формат хранения Big Data в экосистеме Apache Hadoop. Он совместим с большинством сред обработки больших данных в среде Apache Hadoop и похож на другие колоночные форматы файлов: RCFile и Parquet.  
  
<img src="https://github.com/oldos-orwell/data_notes/blob/main/images/orc2.png"/>  

* File Header - содержит магическое число ORC, что позволяет нам прочитать его как ORC-файл.
* Postscript - содержит не все метаданные, а только информацию для интерпретации чтения всего файла. В том числе длину метаданных, длину колонтитула, тип сжатия и тип версии.
* File Footer - содержит все метаданные, что и Parquet, но уже в сжатом виде, чтобы сэкономить крупицу места на дисках за счёт сжатия. В то же время это позволяет ORC быстро отсеивать ненужные файлы. В File Footer хранятся имена колонок, названия, количество строк каждого страйпа, статистика, информация о каждом страйпе, а ещё словари и многое другое.
* Stripe - это аналог Row-group в Parquet, то есть просто разбитие одного большого файла на множество маленьких внутри, невидимое для глаз простого человека. В свою очередь stripe’ы делятся на блоки:
  * Index Data в свою очередь содержит метаинформацию конкретного столбца и конкретного stripe’а, то есть такие данные: как минимум, максимум, сумму и количество строк в конкретном столбце.
  * Row Data - сами данные - физическое разделение на колонки, что позволяет прочитать один маленький блок конкретного страйпа.
    * Columns - колонки, в отличие от Parquet, делятся на блоки индексов и блоки данных. В каждом блоке индексов содержится метаинформация для прочтения конкретного небольшого блока. Этот небольшой блок по умолчанию равен 10000 элементов, что позволяет не грузить лишний раз жёсткий диск и процессор для того, чтобы десериализовать данные.
  * Stripe Footer — это аналог File Footer. Он содержит все метаданные для конкретного stripe’а и позволяет прочитать его очень быстро.   
  
Поддержка ACID в ORC  
Это не та поддержка, которая есть в реляционках, хотя очень похоже. В случае с ORC, когда мы хотим заменить какие-то данные в таблице, он создаёт дополнительный файл, в котором сказано, какое новое значение у определённой строки. Чтобы их каждый раз не перезаписывать, при необходимости поменять 100-200-300 значений  можно добавить новые значения итеративно. Если файлов станет так много, что они будут тормозить систему, то тогда можно их перезаписать.
  
## Parquet vs ORC  
|                            | Parquet                                    | ORC            |
|----------------------------|--------------------------------------------|----------------|
| Чтение по столбцам         | +                                          | +              |
| Статистика по столбцам     | +                                          | +              |
| Изменение порядка столбцов | +                                          |                |
| Добавление столбцов        | +                                          | +              |
| Сжатие                     | поддерживает более слабые алгоритмы сжатия | сжимает лучше  |
| Скорость доступа           | Медленнее                                  | Быстрее        |
| Другое                     | поддержка спецсимволов в именах            | поддержка ACID |
  
В любом случае, выбирать формат хранения нужно в зависимости от движка. Например, для Trino лучше подходит Parquet: For Presto Parquet, it is most efficient when it comes to storage and performance. ORC on the other hand is ideal for storing compact data and skipping over irrelevant data without complex or manually maintained indices. For example, ORC is typically better suited for dimension tables which are slightly smaller while Parquet works better for the fact tables, which are much bigger.  
___
[Форматы файлов в больших данных: краткий ликбез / Хабр](https://habr.com/ru/companies/vk/articles/504952/)  
[Что такое формат данных Apache Parquet для Big Data файлов](https://bigdataschool.ru/wiki/parquet)  
[Как использовать Parquet и не поскользнуться / Хабр](https://habr.com/ru/companies/wrike/articles/279797/)  
[Форматы ORC и Parquet на базе HDFS / Хабр](https://habr.com/ru/companies/oleg-bunin/articles/761780/)  
[Что такое формат Big Data файлов Apache ORC в сравнении с Parquet](https://bigdataschool.ru/wiki/orc)   
[Use ORC Versus Parquet When Using Presto | Ahana](https://ahana.io/answers/when-should-i-use-orc-versus-parquet-when-using-presto/) 