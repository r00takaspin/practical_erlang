## ETS таблицы

И, наконец, в теме KV-структур данных особо мощная магия.
Это не просто структура данных. ETS-таблицу можно рассматривать как базу
данных.  Она рассчитана на хранение большого объема данных и быстрый
доступ к ним.

[ETS](http://www.erlang.org/doc/man/ets.html) таблицы можно поставить в один ряд с такими базами данных как [Memcached](http://memcached.org/) и [Redis](http://redis.io/).
ETS даже лучше, потому что к ним не нужно обращаться по сети,
а данные хранятся прямо в памяти виртуальной машины.

ETS означает Erlang Term Storage. Они реализованы на С как часть
виртуальной машины, и очень эффективны по производительности.  Ради
этой эффективности пришлось пожертвовать некоторыми принципами языка.
Их реализация -- это отдельный императивный мир внутри функционального
языка, с модифицируемыми данными и разделяемой между процессами памятью.
Впрочем, эрланг-разработчик может не беспокоиться об особенностях их
реализации, а просто пользоваться ими.

ETS таблицы хранят кортежи произвольного размера, один из элементов
которых используется как ключ.  По умолчанию -- это первый элемент.
Но при создании ETS можно указать другую позицию элемента-ключа.  И это
нужно делать, если мы будем хранить в таблице records.


### CRUD API

Начнем, как обычно, с создания таблицы и CRUD операций.

```erlang
1> MyEts = ets:new(my_ets, []).
16400
```

Мы создали таблицу с именем **my_ets** и настройками по умолчанию.
В ответ получили идентификатор таблицы.

Кстати, если мы создали ее прямо в консоли, то нужно быть осторожными.
Таблица связана с процессом, который ее создал. И она удаляется, если
родительский процесс завершается. А процесс консоли завершается и
стартует заново при любой ошибке. Так что, если при работе в консоли
допустить опечатку или какое-либо исключение, то таблица исчезнет.

Добавлять можно один элемент, или сразу список элементов:

```erlang
2> ets:insert(MyEts, {1, "Bob", 25}).
true
3> ets:insert(MyEts, [{2, "Bill", 30}, {3, "Helen", 22}]).
true
```

**ets:lookup/2** всегда возвращает список значений, даже если значение
только одно. Таблицы разных типов могут иметь одно или больше значений
для данного ключа, а API для всех типов одинаковое.  Если значения
нет, возвращается пустой список:

```erlang
4> ets:lookup(MyEts, 1).
[{1,"Bob",25}]
5> ets:lookup(MyEts, 3).
[{3,"Helen",22}]
6> ets:lookup(MyEts, 4).
[]
```

Изменение значения тоже делается функцией **ets:insert/2**:

```erlang
7> ets:insert(MyEts, {3, "Helen A.", 21}).
true
8> ets:lookup(MyEts, 3).
[{3,"Helen A.",21}]
```
Ну а удаление функцией **ets:delete/2**:

```erlang
9> ets:delete(MyEts, 2).
true
10> ets:lookup(MyEts, 2).
[]
```

CRUD API довольно простое. В отличие от других модулей, тут всего 3 функции:
**ets:insert/2**, **ets:lookup/2**, **ets:delete/2**.


### Настройки таблицы

Первая настройка, которая нас интересует -- тип таблицы.
Есть 4 типа:

 - **set** -- все ключи должны быть уникальны;
 - **ordered_set** -- ключи должны быть уникальны, и кортежи хранятся в сортированном виде;
 - **bag** -- разрешаются кортежи с одинаковыми ключами, но в целом кортежи должны быть разными;
 - **duplicate_bag** -- разрешаются идентичные кортежи.

Тип таблицы по умолчанию -- **set**.

Далее, можно указать тип доступа к таблице:

 - **public** -- любой процесс может писать в таблицу и читать из нее;
 - **protected** -- любой процесс может читать из таблицы, но писать в нее может только процесс-владелец;
 - **private** -- только процесс-владелец может читать и писать.

Тип доступа по умолчанию -- **protected**.

Настройка, которая указывает позицию ключа в кортеже: **{keypos, K}**.
Вот так создается таблица, в которой планируется хранить records:

```erlang
MyEts = ets:new(my_ets, [set, private, {keypos, 2}]).
```

Есть и другие настройки, но для начала достаточно знать эти.


### Обход таблицы

Кроме получения значения по ключу, ETS-таблицы предоставляют и другие
варианты доступа к данным.  Один из вариантов -- последовательный
обход таблицы от начала к концу, или от конца к началу.

```erlang
1> T = ets:new(my_ets, []).
16400
2> ets:insert(T, [{1,"Bob",25}, {2, "Bill", 18}, {3, "Helen", 20}, {4, "Kate", 25}]).
true
3> Key1 = ets:first(T).
3
4> Key2 = ets:next(T, Key1).
1
5> Key3 = ets:next(T, Key2).
4
6> Key4 = ets:next(T, Key3).
2
7> Key5 = ets:next(T, Key4).
'$end_of_table'
8> K1 = ets:last(T).
3
9> K2 = ets:prev(T, K1).
1
10> K3 = ets:prev(T, K2).
4
11> K4 = ets:prev(T, K3).
2
12> K5 = ets:prev(T, K4).
'$end_of_table'
```

Вызовы **first/1**, **next/2** или **last/1**, **prev/2** возвращают ключи либо атом
**'$end\_of\_table'**, когда достигнут конец таблицы.

Еще есть функция **ets:tab2list/1**, которая возвращает список всех
кортежей, хранящихся в таблице. Однако учитывая, что таблицы могут
содержать очень большой объем данных, разумнее использовать
**first/next** или **last/prev**.


### Выбор объектов по шаблону

Наконец, мы подходим к самым клевым фичам ETS :)

Хорошая новость -- мы можем использовать сопоставление с образцом
(pattern matching), чтобы выбирать нужные данные из таблицы.
Плохая новость -- синтаксис шаблонов отличается от обычного.

Тут есть два варианта: простые шаблоны с ограниченными возможностями,
которые используются в **ets:match/2**; или сложные шаблоны с широкими
возможностями, которые используются в **ets:select/2**.

#### Начнем с ets:match/2

Для начала создадим таблицу:

```erlang
1> T = ets:new(my_ets, []).
16400
2> ets:insert(T, [{1, "Bob", male}, {2, "Helen", female}, {3, "Bill", male}, {4, "Kate", female}]).
true
```

Шаблон представляет собой кортеж. Он содержит либо атомы вида
**'$1'**, **'$2'**, **'$3'**, либо конкретные значения. Атомы
обозначают поля, которые мы хотим извлечь из кортежей.  А значения
должны совпасть с данными в таблице:

```erlang
3> ets:match(T, {'$1', '$2', male}).
[[1,"Bob"],[3,"Bill"]]
4> ets:match(T, {'$1', '$2', female}).
[[2,"Helen"],[4,"Kate"]]
```

Здесь с помощью шаблона **{'$1', '$2', male}** извлекаем Id и имя
пользователя, а атомом **male** ограничиваем выборку. Аналогично
действует второй шаблон.

Еще можно использовать атом **'_'**, который совпадает с любым значением:

```erlang
5> ets:match(T, {'$2', '$1', '_'}).
[["Helen",2],["Kate",4],["Bob",1],["Bill",3]]
```
Шаблон **{'$2', '$1', '_'}** совпадет со всеми значениями в таблице.
Обратите внимание, что мы сперва указали '$2', а затем '$1'.
И в результате получили на первом месте имя, на втором Id.

Если мы хотим извлечь не отдельные поля из кортежей, а кортежи
целиком, то используем **ets:match_object/2**:

```erlang
6> ets:match_object(T, {'$1', '_', male}).
[{1,"Bob",male},{3,"Bill",male}]
```

Еще есть функция **ets:match_delete/2**, которая удаляет из таблицы
элементы, совпавшие с шаблоном.


#### Рассмотрим ets:select/2

Эта функция использует более сложные шаблоны. Это даже не шаблоны, а
отдельный язык со своим синтаксисом. Язык называется **спецификация
совпадения** (match specification).

Плохая новость -- на этом языке очень неудобно читать и
писать. Хорошая новость -- этого не нужно делать, потому что есть
синтаксический сахар.

Сперва разберем шаблон как он есть. [Пример взят из книги Фреда Хеберта](http://learnyousomeerlang.com/ets#you-have-been-selected).

```erlang
[
{{'$1','$2',<<1>>,'$3','$4'},
[{'andalso',{'>','$4',150},{'<','$4',500}},
{'orelse',{'==','$2',meat},{'==','$2',dairy}}],
['$1']},
{{'$1','$2',<<1>>,'$3','$4'},
[{'<','$3',4.0},{is_float,'$3'}],
['$1']}
]
```

Здесь список из двух шаблонов. Разберем каждый из них по частям.

Первая часть называется **базовый шаблон** (Initial Pattern). Он такой же,
какой используется в **ets:match/2**.

```erlang
{'$1','$2',<<1>>,'$3','$4'}
```

Мы ищем совпадение с кортежем из 5-ти элементов.  В этом кортеже мы
связываем 1-й, 2-й, 4-й, и 5-й элементы с переменными '$1', '$2' и
т.д. А 3-й элемент должен иметь значение &lt;&lt;1&gt;&gt;.

Вторая часть -- **гарды**.

```erlang
[{'andalso',{'>','$4',150},{'<','$4',500}},
{'orelse',{'==','$2',meat},{'==','$2',dairy}}]
```

Те, кто знаком с языком Lisp сразу поняли этот синтаксис.  Для
остальных требуется пояснение.  Здесь используется префиксная
нотация. Вместо привычного *аргумент1, операция, аргумент2*, пишется
*операция, аргумент1, аргумент2*.

В данном случае гарды читаются так:
Переменная '$4' должна быть больше 150 и меньше 500,
переменная '$2' должна иметь значение meat либо dairy.

Третья часть -- это значение, которое мы хотим вернуть. В данном
случае значение переменной '$1'.

```erlang
['$1']
```

Второй шаблон имеет такую же базу, такое же возвращаемое значение, но
другие гарды. Переменная '$3' должна быть float и иметь значение
меньше 4.0.

```erlang
{{'$1','$2',<<1>>,'$3','$4'},
[{'<','$3',4.0},{is_float,'$3'}],
['$1']}
```

Любители Lisp могут пользоваться этим языком, а мы рассмотрим
синтаксический сахар. Шаблон можно записать в синтаксисе, очень похожем
на обычный эрланг:

```erlang
fun({Food, Type, <<1>>, Price, Calories})
    when Calories > 150 andalso Calories < 500,
         Type == meat orelse Type == dairy;
         Price < 4.00, is_float(Price) ->
    Food
end.
```

Этот код выглядит как анонимная функция, которая принимает кортеж из
5-ти элементов, применяет гарды, и возвращает значение. На самом деле
это не функция, а шаблон, который с помощью **ets:fun2ms/1** (fun to
match specification) преобразуется в то, что мы разбирали выше.

Как видно, этот код читать гораздо легче. Во-первых, потому что
переменные именованы, а не просто циферки. Во-вторых потому, что это
обычный синтаксис Эрланг.

Теперь мы можем перейти к примерам. Сделаем такой модуль:


```erlang
-module(main).
-include_lib("stdlib/include/ms_transform.hrl").
-export([init/0, select/0, select/1]).
init() ->
    ets:new(my_ets, [named_table]),
    ets:insert(my_ets, [{1, "Bob", 25, male},
                        {2, "Helen", 17, female},
                        {3, "Bill", 28, male},
                        {4, "Kate", 22, female},
                        {5, "Ivan", 14, male}]),
    ok.
select() ->
    MS = ets:fun2ms(fun({Id, Name, Age, Gender})
                          when Age >= 17 andalso Gender =:= male ->
                            [Id, Name]
                    end),
    select(MS).
select(MS) ->
    ets:select(my_ets, MS).
```

Чтобы **ets:fun2ms/1** работал в модуле нужно подключить заголовочный
файл ms_transform.hrl.

Таблицу создадим с опцией **named_table**, это позволит обращаться к ней
по имени, а не по идентификатору. Сразу заполним таблицу 5-ю кортежами,
представляющих пользователей.

Прямо в модуле реализуем выборку по шаблону: вернуть Id и Name для
всех пользователей мужского пола старше 17.

И попробуем это все в консоли:

```erlang
1> c(main).
{ok,main}
2> main:init().
ok
3> main:select().
[[1,"Bob"],[3,"Bill"]]
```

Шаблон работает. Попробуем из консоли задать другой шаблон:

```erlang
4> MS = ets:fun2ms(fun(User = {Id, Name, Age, Gender}) when Age > 20 -> User end).
[{{'$1','$2','$3','$4'},[{'>','$3',20}],['$_']}]
5> main:select(MS).
[{4,"Kate",22,female},{1,"Bob",25,male},{3,"Bill",28,male}]
```

Выбираем всех пользователей старше 20 лет. В консоли видно, во что
скомпилировался шаблон.

```erlang
[{{'$1','$2','$3','$4'},[{'>','$3',20}],['$_']}]
```

Здесь обратите внимание, что с помощью переменной **'$_'** мы возвращаем
весь кортеж, а не отдельные поля из него.


### Что еще нужно знать о ETS таблицах

ETS таблицы не подвергаются сборке мусора. Программист сам должен
следить за тем, чтобы удалять из таблиц ненужные данные. Иначе,
постоянно добавляя новые кортежи, но не удаляя ничего из таблицы,
он получит утечку памяти.

Процесс, в котором таблица создана, является ее владельцем. Если
процесс завершится (неважно, нормально или из-за ошибки), таблица
будет удалена. Владельца можно динамически менять, передавая
ответственность за таблицу другому процессу.

Вызовы **ets:insert/2** и **ets:delete/2** выполняются атомарно и
изолированно.  Атомарно -- это значит, что операция либо завершится
успешно, либо будет отменена, промежуточные состояния не останутся в
таблице.  Изолированно -- это значит, что другие процессы при чтении
из таблицы не будут видеть промежуточных состояний.

При обходе таблицы с помощью **first/next** (**last/prev**) и при
выборке с помощью **match/select** гарантируется, что каждый кортеж
будет рассмотрен, и рассмотрен только один раз.  Но если во время
обхода/выборки будут вставлены новые кортежи, то они могут быть
рассмотрены, а могут быть пропущены. Для них гарантий нет.


### DETS и Mnesia

ETS -- это in-memory база данных. То есть, она хранит данные только в
оперативной памяти. Конечно, есть возможность хранить данные и на диске.
Для этого используется [DETS (Disk EST)](http://www.erlang.org/doc/man/dets.html).

Модуль DETS предлагает аналогичный API, как и ETS. По понятным причинам
он работает медленнее.

А поверх DETS построена полноценная база данных [Mnesia](http://www.erlang.org/doc/man/mnesia.html).
Она распределенная, то есть, создает общее хранилище для нескольких
узлов в кластере.  Она поддерживает сложные запросы и транзакции.

Все бы хорошо, но у DETS и Mnesia есть пара неприятных особенностей.

Во-первых, DETS не может хранить больше 2Гб данных. Это ограничивает
их применение, т.к. часто нужно хранить гораздо большие объемы данных --
десятки и сотни гигабайт.

Во-вторых, если DETS падает с ошибкой, то восстановление данных может
занять очень много времени. Крайне неприятно, что запуск ноды после
аварии занимает несколько десятков минут, а то и несколько часов. И
все это время ваш сервер не может обслуживать клиентов.

Из-за этого большинство эрланг программистов предпочитают использовать
отдельную базу данных. Либо традиционные PostgreSQL, MySQL, либо
различные NoSQL решения.
