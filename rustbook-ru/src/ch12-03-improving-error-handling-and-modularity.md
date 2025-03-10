## Рефакторинг для улучшения модульности и обработки ошибок

Для улучшения программы мы исправим 4 имеющихся проблемы, связанных со структурой программы и тем как обрабатываются потенциальные ошибки. Во-первых, функция `main` на данный момент решает две задачи:  анализирует переменные командной строки и читает файлы. По мере роста программы количество отдельных задач, которые обрабатывает функция `main`, будет увеличиваться. Поскольку эта функция получает больше обязанностей, то становится все труднее понимать её, труднее тестировать и труднее изменять, не сломав одну из её частей. Лучше всего разделить функциональность, чтобы каждая функция отвечала за одну задачу.

Эта проблема также связана со второй проблемой: хотя переменные `query` и `file_path` являются переменными конфигурации нашей программы, переменные типа `contents` используются для выполнения логики программы. Чем длиннее становится `main`, тем больше переменных нам нужно будет добавить в область видимости; чем больше у нас переменных в области видимости, тем сложнее будет отслеживать назначение каждой переменной. Лучше всего сгруппировать переменные конфигурации в одну структуру, чтобы сделать их назначение понятным.

Третья проблема заключается в том, что мы используем `expect` для вывода информации об ошибке при проблеме с чтением файла, но сообщение об ошибке просто выведет текст`Should have been able to read the file`. Чтение файла может не сработать по разным причинам, например: файл не найден или у нас может не быть разрешения на его чтение. Сейчас же, независимо от ситуации, мы напечатаем одно и то же сообщение об ошибке, что не даст пользователю никакой информации!

В-четвёртых, мы используем `expect` неоднократно для обработки различных ошибок и если пользователь запускает нашу программу без указания достаточного количества аргументов он получит ошибку `index out of bounds` из Rust, что не совсем понятно описывает проблему. Было бы лучше, если бы весь код обработки ошибок находился в одном месте, чтобы тем, кто будет поддерживать наш код в дальнейшем, нужно было бы вносить изменения только здесь, если потребуется изменить логику обработки ошибок. Наличие всего кода обработки ошибок в одном месте гарантирует, что мы напечатаем сообщения, которые будут иметь смысл для наших конечных пользователей.

Давайте решим эти четыре проблемы путём рефакторинга нашего проекта.

### <a name="separation-of-concerns-for-binary-projects"></a>Разделение ответственности для бинарных проектов

Организационная проблема распределения ответственности за выполнение нескольких задач функции `main`  является общей для многих бинарных проектов. В результате Rust сообщество разработало процесс для использования в качестве руководства по разделению ответственности бинарной программы, когда код в `main` начинает увеличиваться. Процесс имеет следующие шаги:

- Разделите код программы на два файла *main.rs* и *lib.rs*. Перенесите всю логику работы программы в файл *lib.rs*.
- Пока ваша логика синтаксического анализа командной строки мала, она может оставаться в файле *main.rs*.
- Когда логика синтаксического анализа командной строки становится сложной, извлеките её из *main.rs* и переместите в *lib.rs.*

Функциональные обязанности, которые остаются в функции `main` после этого процесса должно быть ограничено следующим:

- Вызов логики разбора командной строки со значениями аргументов
- Настройка любой другой конфигурации
- Вызов функции `run` в *lib.rs*
- Обработка ошибки, если `run` возвращает ошибку

Этот шаблон о разделении ответственности: *main.rs* занимается запуском программы, а *lib.rs* обрабатывает всю логику задачи. Поскольку нельзя проверить функцию `main` напрямую, то такая структура позволяет проверить всю логику программы путём перемещения её в функции внутри *lib.rs*. Единственный код, который остаётся в *main.rs* будет достаточно маленьким, чтобы проверить его корректность прочитав код. Давайте переработаем нашу программу, следуя этому процессу.

#### Извлечение парсера аргументов

Мы извлечём функциональность для разбора аргументов в функцию, которую вызовет `main` для подготовки к перемещению логики разбора командной строки в файл *src/lib.rs*. Листинг 12-5 показывает новый запуск `main`, который вызывает новую функцию `parse_config`, которую мы определим сначала в *src/main.rs.*

<span class="filename">Файл: src/main.rs</span>

```rust,ignore
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-05/src/main.rs:here}}
```

<span class="caption">Листинг 12-5. Извлечение функции <code>parse_config</code> из <code>main</code></span>

Мы все ещё собираем аргументы командной строки в вектор, но вместо присваивания значение аргумента с индексом 1 переменной `query` и значение аргумента с индексом 2 переменной с именем `file_path` в функции `main`, мы передаём весь вектор в функцию `parse_config`. Функция `parse_config` затем содержит логику, которая определяет, какой аргумент идёт в какую переменную и передаёт значения обратно в `main`. Мы все ещё создаём переменные `query` и `file_path` в `main`, но `main` больше не несёт ответственности за определение соответствия аргумента командной строки и соответствующей переменной.

Эта доработка может показаться излишней для нашей маленькой программы, но мы проводим рефакторинг небольшими, постепенными шагами. После внесения этого изменения снова запустите программу и убедитесь, что анализ аргументов все ещё работает. Также хорошо часто проверять прогресс, чтобы помочь определить причину проблем, когда они возникают.

#### Группировка конфигурационных переменных

Мы можем сделать ещё один маленький шаг для улучшения функции `parse_config`. На данный момент мы возвращаем кортеж, но затем мы немедленно разделяем его снова на отдельные части. Это признак того, что, возможно,  пока у нас нет правильной абстракции.

Ещё один индикатор, который показывает, что есть место для улучшения, это часть `config` из `parse_config`, что подразумевает, что два значения, которые мы возвращаем, связаны друг с другом и оба являются частью одного конфигурационного значения. В настоящее время мы не отражаем этого смысла в структуре данных, кроме группировки двух значений в кортеж; мы могли бы поместить оба значения в одну структуру и дать каждому из полей структуры понятное имя. Это облегчит будущую поддержку этого кода, чтобы понять, как различные значения относятся друг к другу и какое их назначение.

В листинге 12-6 показаны улучшения функции `parse_config` .

<span class="filename">Файл: src/main.rs</span>

```rust,should_panic,noplayground
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-06/src/main.rs:here}}
```

<span class="caption">Листинг 12-6: Рефакторинг функции <code>parse_config</code>, чтобы возвращать экземпляр структуры <code>Config</code></span>

Мы добавили структуру с именем `Config` объявленную с полями назваными как `query` и `file_path`. Сигнатура `parse_config` теперь указывает, что она возвращает значение `Config`. В теле `parse_config`, где мы возвращали срезы строк, которые ссылаются на значения `String` в `args`, теперь мы определяем `Config` как содержащие собственные `String` значения. Переменная `args` в `main` является владельцем значений аргумента и позволяют функции `parse_config` только одалживать их, что означает, что мы бы нарушили правила заимствования Rust, если бы `Config` попытался бы взять во владение значения в `args` .

Мы можем управлять данными `String` разным количеством способов, но самый простой, хотя и отчасти неэффективный это вызвать метод `clone` у значений. Он сделает полную копию данных для экземпляра `Config` для владения, что занимает больше времени и памяти, чем сохранение ссылки на строку данных. Однако клонирование данных также делает наш код очень простым, потому что нам не нужно управлять временем жизни ссылок; в этом обстоятельстве, отказ от небольшой производительности, чтобы получить простоту, стоит небольших компромисса.

>  <h>Компромиссы при использовании метода <code>clone</code></h>Существует тенденция в среде программистов Rust избегать использования `clone`, т.к. это понижает эффективность работы кода. В [Главе 13]<!-- ignore -->, вы изучите более эффективные методы, которые могут подойти в подобной ситуации. Но сейчас можно копировать несколько строк, чтобы продолжить работу, потому что вы сделаете эти копии только один раз, а ваше имя файла и строка запроса будут очень маленькими. Лучше иметь работающую программу, которая немного неэффективна, чем пытаться заранее оптимизировать код при первом написании. По мере приобретения опыта работы с Rust вам будет проще начать с наиболее эффективного решения, но сейчас вполне приемлемо вызвать `clone`.
>

Мы обновили код в `main` поэтому он помещает экземпляр `Config` возвращённый из `parse_config` в переменную с именем `config`, и мы обновили код, в котором ранее использовались отдельные переменные `query` и `file_path`, так что теперь он использует вместо этого поля в структуре `Config`.

Теперь наш код более чётко передаёт то, что `query` и `file_path` связаны и что цель из использования состоит в том, чтобы настроить, как программа будет работать. Любой код, который использует эти значения знает, что может найти их в именованных полях экземпляра `config` по их назначению.

#### Создание конструктора для структуры `Config`

Пока что мы извлекли логику, отвечающую за синтаксический анализ аргументов командной строки из `main` и поместили его в функцию `parse_config`. Это помогло нам увидеть, что значения `query` и `file_path` были связаны и что их отношения должны быть отражены в нашем коде. Затем мы добавили структуру `Config` в качестве названия связанных общей целью `query` и `file_path` и чтобы иметь возможность вернуть именованные значения как имена полей структуры из функции `parse_config`.

Итак, теперь целью функции `parse_config` является создание экземпляра `Config`, мы можем изменить `parse_config` из простой функции на функцию названную `new`, которая связана со структурой `Config`. Выполняя это изменение мы сделаем код более идиоматичным. Можно создавать экземпляры типов в стандартной библиотеке, такие как `String` с помощью вызова `String::new`. Точно так же изменив название `parse_config` на название функции `new`, связанную с `Config`, мы будем уметь создавать экземпляры `Config`, вызывая `Config::new`. Листинг 12-7 показывает изменения, которые мы должны сделать.

<span class="filename">Файл: src/main.rs</span>

```rust,should_panic,noplayground
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-07/src/main.rs:here}}
```

<span class="caption">Листинг 12-7. Изменение имени с <code>parse_config</code> на <code>Config::new</code></span>

Мы обновили `main` где вызывали `parse_config`, чтобы вместо этого вызывалась `Config::new`. Мы изменили имя `parse_config` на `new` и перенесли его внутрь блока `impl`, который связывает функцию `new` с `Config`. Попробуйте снова скомпилировать код, чтобы убедиться, что он работает.

### Исправление ошибок обработки

Теперь мы поработаем над исправлением обработки ошибок. Напомним, что попытки получить доступ к значениям в векторе `args` с индексом 1 или индексом 2 приведут к панике, если вектор содержит менее трёх элементов. Попробуйте запустить программу без каких-либо аргументов; это будет выглядеть так:

```console
{{#include ../listings/ch12-an-io-project/listing-12-07/output.txt}}
```

Строка `index out of bounds: the len is 1 but the index is 1` является сообщением об ошибке предназначенной для программистов. Она не поможет нашим конечным пользователям понять, что случилось и что они должны сделать вместо этого. Давайте исправим это сейчас.

#### Улучшение сообщения об ошибке

В листинге 12-8 мы добавляем проверку в функцию `new`, которая будет проверять, что срез достаточно длинный, перед попыткой доступа по индексам 1 и 2. Если срез не достаточно длинный, программа паникует и отображает улучшенное сообщение об ошибке.

<span class="filename">Файл: src/main.rs</span>

```rust,ignore
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-08/src/main.rs:here}}
```

<span class="caption">Листинг 12-8. Добавление проверки на число аргументы</span>

Этот код похож на [функцию `Guess::new` написанную в листинге 9-13], где мы вызывали `panic!`, когда `value` аргумента вышло за пределы допустимых значений. Здесь вместо проверки на диапазон значений, мы проверяем, что длина `args` не менее 3 и остальная часть функции может работать при условии, что это условие было выполнено. Если в `args` меньше трёх элементов, это условие будет истинным и мы вызываем макрос `panic!` для немедленного завершения программы.

Имея нескольких лишних строк кода в `new`, давайте запустим программу снова без аргументов, чтобы увидеть, как выглядит ошибка:

```console
{{#include ../listings/ch12-an-io-project/listing-12-08/output.txt}}
```

Этот вывод лучше: у нас теперь есть разумное сообщение об ошибке. Тем не менее, мы также имеем постороннюю информацию, которую мы не хотим предоставлять нашим пользователям. Возможно, использованная техника, которую мы использовали в листинге 9-13, не является лучшей для использования: вызов `panic!` больше подходит для программирования проблемы, чем решения проблемы, [как обсуждалось в главе 9]<!-- ignore -->. Вместо этого мы можем использовать другую технику, о которой вы узнали в главе 9 [возвращая `Result`]<!-- ignore -->, которая указывает либо на успех, либо на ошибку.

<!-- Old headings. Do not remove or links may break. -->

<a id="returning-a-result-from-new-instead-of-calling-panic"></a>

#### Возвращение `Result` из вместо вызова `panic!`

Мы можем вернуть значение `Result`, которое будет содержать экземпляр `Config` в успешном случае и опишет проблему в случае ошибки. Мы так же изменим функцию `new` на `build` потому что многие программисты ожидают что `new` никогда не завершится неудачей. Когда `Config::build` взаимодействует с `main`, мы можем использовать тип `Result` как сигнал возникновения проблемы. Затем мы можем изменить `main`, чтобы преобразовать вариант `Err` в более практичную ошибку для наших пользователей без окружающего текста вроде `thread 'main'` и `RUST_BACKTRACE`, что происходит при вызове `panic!`.

Листинг 12-9 показывает изменения, которые нужно внести в возвращаемое значения функции `Config::build`, и в тело функции, необходимые для возврата типа `Result`. Заметьте, что этот код не скомпилируется, пока мы не обновим `main`, что мы и сделаем в следующем листинге.

<span class="filename">Файл: src/main.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-09/src/main.rs:here}}
```

<span class="caption">Листинг 12-9. Возвращение типа <code>Result</code> из <code>Config::build</code></span>

Наша функция `build` теперь возвращает `Result` с экземпляром `Config` в случае успеха и `&'static str` в случае ошибки. Значения ошибок всегда будут строковыми литералами, которые имеют время жизни `'static`.

Мы внесли два изменения в тело функции `build`: вместо вызова `panic!`, когда пользователь не передаёт достаточно аргументов, мы теперь возвращаем `Err` значение и мы завернули возвращаемое значение `Config` в <code>Ok</code> . Эти изменения заставят функцию соответствовать своей новой сигнатуре типа.

Возвращение значения `Err` из `Config::build` позволяет функции `main` обработать значение `Result` возвращённое из функции `build` и выйти из процесса более чисто в случае ошибки.

<!-- Old headings. Do not remove or links may break. -->

<a id="calling-confignew-and-handling-errors"></a>

#### Вызов `Config::build` и обработка ошибок

Чтобы обработать ошибку и вывести более дружественное сообщение об ошибке, нам нужно обновить код `main` для обработки `Result`, возвращаемого из `Config::build` как показано в листинге 12-10. Мы также возьмём на себя ответственность за выход из программы командной строки с ненулевым кодом ошибки `panic!` и реализуем это вручную. Не нулевой статус выхода - это соглашение, которое сигнализирует процессу, который вызывает нашу программу, что программа завершилась с ошибкой.

<span class="filename">Файл: src/main.rs</span>

```rust,ignore
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-10/src/main.rs:here}}
```

<span class="caption">Листинг 12-10. Выход с кодом ошибки если создание новой <code>Config</code> терпит неудачу</span>

В этом листинге мы использовали метод, который мы ещё не рассматривали детально: `unwrap_or_else`, который в стандартной библиотеке определён как `Result<T, E>`. Использование `unwrap_or_else` позволяет нам определить некоторые пользовательские  ошибка обработки, не содержащие `panic!`. Если `Result` является значением `Ok`, поведение этого метода аналогично `unwrap`: возвращает внутреннее значение из обёртки `Ok`. Однако, если значение значение `Err`, то этот метод вызывает код *замыкания*, которое является анонимной функцией определённую заранее и передаваемую в качестве аргумента в `unwrap_or_else`. Мы рассмотрим замыкания более подробно в [главе 13](ch13-00-functional-features.html). В данный момент, вам просто нужно знать, что `unwrap_or_else` передаст внутреннее значение `Err`, которое в этом случае является статической строкой `not enough arguments`, которое мы добавили в листинге 12-9, в наше замыкание как аргумент `err` указанное между вертикальными линиями. Код в замыкании может затем использовать значение `err` при выполнении.

Мы добавили новую строку `use`, чтобы подключить `process` из стандартной библиотеки в область видимости. Код в замыкании, который будет запущен в случае ошибки содержит только две строчки: мы печатаем значение `err` и затем вызываем `process::exit`. Функция `process::exit` немедленно остановит программу и вернёт номер, который был передан в качестве кода состояния выхода. Это похоже на обработку с помощью макроса `panic!`, которую мы использовали в листинге 12-8, но мы больше не получаем весь дополнительный вывод. Давай попробуем:

```console
{{#include ../listings/ch12-an-io-project/listing-12-10/output.txt}}
```

Замечательно! Этот вывод намного дружелюбнее для наших пользователей.

### Извлечение логики из `main`

Теперь, когда мы закончили рефакторинг разбора конфигурации, давайте обратимся к логике программы. Как мы указали в разделе [«Разделение ответственности в бинарных проектах»](#separation-of-concerns-for-binary-projects)<!-- ignore -->, мы извлечём функцию с именем `run`, которая будет содержать всю логику, присутствующую в настоящее время в функции `main` и которая не связана с настройкой конфигурации или обработкой ошибок. Когда мы закончим, то `main` будет краткой, легко проверяемой и мы сможем написать тесты для всей остальной логики.

Код 12-11 демонстрирует извлечённую логику в функцию `run`. Мы делаем маленькое, инкрементальное приближение к извлечению функции. Код всё ещё сосредоточен в файле *src/main.rs*:

<span class="filename">Файл: src/main.rs</span>

```rust,ignore
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-11/src/main.rs:here}}
```

<span class="caption">Листинг 12-11. Извлечение функции <code>run</code>, содержащей остальную логику программы</span>

Функция `run` теперь содержит всю оставшуюся логику из `main`, начиная от чтения файла. Функция `run` принимает экземпляр `Config` как аргумент.

#### Возврат ошибок из функции `run`

Оставшаяся логика программы выделена в функцию `run`, где мы можем улучшить обработку ошибок как мы уже делали с `Config::build` в листинге 12-9. Вместо того, чтобы позволить программе паниковать с помощью вызова `expect`, функция `run` вернёт `Result<T, E>`, если что-то пойдёт не так. Это позволит далее консолидировать логику обработки ошибок в `main` удобным способом. Листинг 12-12 показывает изменения, которые мы должны внести в сигнатуру и тело `run`.

<span class="filename">Файл: src/main.rs</span>

```rust,ignore
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-12/src/main.rs:here}}
```

<span class="caption">Листинг 12-12. Изменение функции <code>run</code> для возврата <code>Result</code></span>

Здесь мы сделали три значительных изменения. Во-первых, мы изменили тип возвращаемого значения функции `run` на `Result<(), Box<dyn Error>>` . Эта функция ранее возвращала тип `()` и мы сохраняли его как значение, возвращаемое в случае `Ok`.

Для типа ошибки мы использовали *объект типаж* `Box<dyn Error>` (и вверху мы подключили тип `std::error::Error` в область видимости с помощью оператора `use`). Мы рассмотрим типажи объектов в [главе 17]<!-- ignore -->. Сейчас просто знайте, что `Box<dyn Error>` означает, что функция будет возвращать тип реализующий типаж `Error`, но не нужно указывать, какой именно будет тип возвращаемого значения. Это даёт возможность возвращать значения ошибок, которые могут быть разных типов в разных случаях. Ключевое слово `dyn` сокращение для слова «динамический».

Во-вторых, мы убрали вызов `expect` в пользу использования оператора `?`, как мы обсудили в [главе 9]<!-- ignore -->. Скорее, чем вызывать `panic!` в случае ошибки, оператор `?` вернёт значение ошибки из текущей функции для вызывающего, чтобы он её обработал.

В-третьих, функция `run` теперь возвращает значение `Ok` в случае успеха. В сигнатуре функции `run` успешный тип объявлен как `()`, который означает, что нам нужно обернуть значение единичного типа в значение `Ok`. Данный синтаксис `Ok(())` поначалу может показаться немного странным, но использование `()` выглядит как идиоматический способ указать, что мы вызываем `run` для его побочных эффектов; он не возвращает значение, которое нам нужно.

Когда вы запустите этот код, он скомпилируется, но отобразит предупреждение:

```console
{{#include ../listings/ch12-an-io-project/listing-12-12/output.txt}}
```

Rust говорит, что наш код проигнорировал `Result` значение и значение `Result` может указывать на то, что произошла ошибка. Но мы не проверяем, была ли ошибка и компилятор напоминает нам, что мы, вероятно, хотели здесь выполнить некоторый код обработки ошибок! Давайте исправим эту проблему сейчас.

#### Обработка ошибок, возвращённых из `run` в `main`

Мы будем проверять и обрабатывать ошибки используя методику, аналогичную той, которую мы использовали для `Config::build` в листинге 12-10, но с небольшой разницей:

<span class="filename">Файл: src/main.rs</span>

```rust,ignore
{{#rustdoc_include ../listings/ch12-an-io-project/no-listing-01-handling-errors-in-main/src/main.rs:here}}
```

Мы используем `if let` вместо `unwrap_or_else` чтобы проверить, возвращает ли `run` значение `Err` и вызывается `process::exit(1)`, если это так. Функция `run` не возвращает значение, которое мы хотим развернуть методом `unwrap`, таким же образом как `Config::build` возвращает экземпляр `Config`. Так как `run` возвращает `()` в случае успеха и мы заботимся только об обнаружении ошибки, то нам не нужно вызывать `unwrap_or_else`, чтобы вернуть развёрнутое значение, потому что оно будет только `()`.

Тело функций `if let` и `unwrap_or_else` одинаковы в обоих случаях: мы печатаем ошибку и выходим.

### Разделение кода на библиотечный крейт

Наш проект `minigrep` пока выглядит хорошо! Теперь мы разделим файл *src/main.rs* и поместим некоторый код в файл *src/lib.rs*. Таким образом мы сможем его тестировать и чтобы в файле *src/main.rs* было меньшее количество функциональных обязанностей.

Давайте перенесём весь код не относящийся к функции `main` из файла *src/main.rs* в новый файл *src/lib.rs*:

- Определение функции `run`.
- Соответствующие инструкции `use`.
- Определение структуры `Config`.
- Определение функции `Config::build`

Содержимое *src/lib.rs* должно иметь сигнатуры, показанные в листинге 12-13 (мы опустили тела функций для краткости). Обратите внимание, что код не будет компилироваться пока мы не изменим *src/main.rs* в листинге 12-14.

<span class="filename">Файл: src/lib.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-13/src/lib.rs:here}}
```

<span class="caption">Листинг 12-13. Перемещение <code>Config</code> и <code>run</code> в <em>src/lib.rs</em></span>

Мы добавили спецификатор доступа `pub` к структуре `Config`, а также её полям, к методу `build` и функции `run`. Теперь у нас есть библиотечный крейт, который содержит публичный API, который мы можем протестировать!

Теперь нам нужно подключить код, который мы переместили в *src/lib.rs,* в область видимости бинарного крейта внутри *src/main.rs*, как показано в листинге 12-14.

<span class="filename">Файл: src/main.rs</span>

```rust,ignore
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-14/src/main.rs:here}}
```

<span class="caption">Листинг 12-14. Использование крейта библиотеки <code>minigrep</code> внутри <em>src/main.rs</em></span>

Мы добавляем `use minigrep::Config` для подключения типа `Config` из крейта библиотеки в область видимости бинарного крейта и добавляем к имени функции `run` префикс нашего крейта. Теперь все функции должны быть подключены и должны работать. Запустите программу с `cargo run` и убедитесь, что все работает правильно.

Уф! Было много работы, но мы настроены на будущий успех. Теперь проще обрабатывать ошибки и мы сделали код более модульным. С этого момента почти вся наша работа будет выполняться внутри *src/lib.rs*.

Давайте воспользуемся этой новой модульностью, сделав что-то, что было бы трудно со старым кодом, но легко с новым кодом: мы напишем несколько тестов!


[Главе 13]: ch13-00-functional-features.html
[функцию `Guess::new` написанную в листинге 9-13]: ch09-03-to-panic-or-not-to-panic.html#creating-custom-types-for-validation
[как обсуждалось в главе 9]: ch09-03-to-panic-or-not-to-panic.html#guidelines-for-error-handling
[главе 17]: ch17-00-oop.html
[главе 9]: ch09-02-recoverable-errors-with-result.html#a-shortcut-for-propagating-errors-the--operator