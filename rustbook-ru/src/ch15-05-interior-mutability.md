## `RefCell<T>` и шаблон внутренней изменяемости

*Внутренняя изменяемость* - это паттерн проектирования Rust, который позволяет вам изменять данные даже при наличии неизменяемых ссылок на эти данные; обычно такое действие запрещено правилами заимствования. Для изменения данных паттерн использует `unsafe` код внутри структуры данных, чтобы обойти обычные правила Rust, регулирующие изменяемость и заимствование. Небезопасный (unsafe) код даёт понять компилятору, что мы самостоятельно следим за соблюдением этих правил, а не полагаемся на то, что компилятор будет делать это для нас; подробнее о небезопасном коде мы поговорим в главе 19.

Мы можем использовать типы, в которых применяется паттерн внутренней изменяемости, только если мы можем гарантировать, что правила заимствования будут соблюдаться во время выполнения, несмотря на то, что компилятор не сможет этого гарантировать. В этом случае `небезопасный` код оборачивается безопасным API, и внешне тип остаётся неизменяемым.

Давайте изучим данную концепцию с помощью типа данных `RefCell<T>`, который реализует этот шаблон.

### Применение правил заимствования во время выполнения с помощью `RefCell<T>`

В отличие от `Rc<T>` тип `RefCell<T>` предоставляет единоличное владение данными, которые он содержит. В чем же отличие типа `RefCell<T>` от `Box<T>`? Давайте вспомним правила заимствования из Главы 4:

- В любой момент времени вы можете иметь *либо* одну изменяемую ссылку либо сколько угодно неизменяемых ссылок (но не оба типа ссылок одновременно).
- Ссылки всегда должны быть действительными.

С помощью ссылок и типа `Box<T>` инварианты правил заимствования применяются на этапе компиляции. С помощью `RefCell<T>` они применяются *во время работы программы*. Если вы нарушите эти правила, работая с ссылками, то будет ошибка компиляции. Если вы работаете с `RefCell<T>` и нарушите эти правила, то программа вызовет панику и завершится.

Преимущества проверки правил заимствования во время компиляции заключаются в том, что ошибки будут обнаруживаться раньше - ещё в процессе разработки, а производительность во время выполнения не пострадает, поскольку весь анализ завершён заранее. По этим причинам проверка правил заимствования во время компиляции является лучшим выбором в большинстве случаев, и именно поэтому она используется в Rust по умолчанию.

Преимущество проверки правил заимствования во время выполнения заключается в том, что определённые сценарии, безопасные для памяти, разрешаются там, где они были бы запрещены проверкой во время компиляции. Статический анализ, как и компилятор Rust, по своей сути консервативен. Некоторые свойства кода невозможно обнаружить, анализируя код: самый известный пример - проблема остановки, которая выходит за рамки этой книги, но является интересной темой для исследования.

Поскольку некоторый анализ невозможен, то если компилятор Rust не может быть уверен, что код соответствует правилам владения, он может отклонить корректную программу; таким образом он является консервативным. Если Rust принял некорректную программу, то пользователи не смогут доверять гарантиям, которые даёт Rust. Однако, если Rust отклонит корректную программу, то программист будет испытывать неудобства, но ничего катастрофического не произойдёт. Тип `RefCell<T>` полезен, когда вы уверены, что ваш код соответствует правилам заимствования, но компилятор не может понять и гарантировать этого.

Подобно типу `Rc<T>`, тип `RefCell<T>` предназначен только для использования в одно поточных сценариях и выдаст ошибку времени компиляции, если вы попытаетесь использовать его в много поточном контексте. Мы поговорим о том, как получить функциональность `RefCell<T>` во много поточной программе в главе 16.

Вот список причин выбора типов `Box<T>`, `Rc<T>` или `RefCell<T>`:

- Тип `Rc<T>` разрешает множественное владение одними и теми же данными; типы `Box<T>` и `RefCell<T>` разрешают иметь единственных владельцев.
- Тип `Box<T>` разрешает неизменяемые или изменяемые владения, проверенные при компиляции; тип `Rc<T>` разрешает только неизменяемые владения, проверенные при компиляции; тип `RefCell<T>` разрешает неизменяемые или изменяемые владения, проверенные во время выполнения.
- Поскольку `RefCell<T>` разрешает изменяемые заимствования, проверенные во время выполнения, можно изменять значение внутри `RefCell<T>` даже если `RefCell<T>` является неизменным.

Изменение значения внутри неизменного значения является шаблоном *внутренней изменяемости* (interior mutability). Давайте посмотрим на ситуацию, в которой внутренняя изменяемость полезна и рассмотрим, как это возможно.

### Внутренняя изменяемость: изменяемое заимствование неизменяемого значения

Следствием правил заимствования является то, что когда у вас есть неизменяемое значение, вы не можете заимствовать его с изменением. Например, этот код не будет компилироваться:

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch15-smart-pointers/no-listing-01-cant-borrow-immutable-as-mutable/src/main.rs}}
```

Если вы попытаетесь скомпилировать этот код, вы получите следующую ошибку:

```console
{{#include ../listings/ch15-smart-pointers/no-listing-01-cant-borrow-immutable-as-mutable/output.txt}}
```

Однако бывают ситуации, в которых было бы полезно, чтобы объект мог изменять себя при помощи своих методов, но казался неизменным для прочего кода. Код вне методов этого объекта не должен иметь возможности изменять его содержимое. Использование `RefCell<T>` - один из способов получить возможность внутренней изменяемости, но при этом `RefCell<T>` не позволяет полностью обойти правила заимствования: средство проверки правил заимствования в компиляторе позволяет эту внутреннюю изменяемость, однако правила заимствования проверяются во время выполнения. Если вы нарушите правила, то вместо ошибки компиляции вы получите `panic!`.

Давайте разберём практический пример, в котором мы можем использовать `RefCell<T>` для изменения неизменяемого значения и посмотрим, почему это полезно.

#### Вариант использования внутренней изменяемости: мок объекты

Иногда во время тестирования программист использует один тип вместо другого для того, чтобы проверить определённое поведение и убедиться, что оно реализовано правильно. Такой тип-заместитель называется *тестовым дублёром*. Воспринимайте его как "каскадёра" в кинематографе, когда дублёр заменяет актёра для выполнения определённой сложной сцены. Тестовые дублёры заменяют другие типы при выполнении тестов. {Инсценировочные (Mock) объекты - это особый тип тестовых дублёров, которые сохраняют данные происходящих во время теста действий тем самым позволяя вам убедиться впоследствии, что все действия были выполнены правильно.

В Rust нет объектов в том же смысле, в каком они есть в других языках и в Rust нет функциональности мок объектов, встроенных в стандартную библиотеку, как в некоторых других языках. Однако вы определённо можете создать структуру, которая будет служить тем же целям, что и мок объект.

Вот сценарий, который мы будем тестировать: мы создадим библиотеку, которая отслеживает значение по отношению к заранее определённому максимальному значению и отправляет сообщения в зависимости от того, насколько текущее значение находится близко к такому максимальному значению. Эта библиотека может использоваться, например, для отслеживания квоты количества вызовов API пользователя, которые ему разрешено делать.

Наша библиотека будет предоставлять только функции отслеживания того, насколько близко к максимальному значению находится значение и какие сообщения должны быть внутри в этот момент. Ожидается, что приложения, использующие нашу библиотеку, предоставят механизм для отправки сообщений: приложение может поместить сообщение в приложение, отправить электронное письмо, отправить текстовое сообщение или что-то ещё. Библиотеке не нужно знать эту деталь. Все что ему нужно - это что-то, что реализует типаж, который мы предоставим с названием `Messenger`. Листинг 15-20 показывает код библиотеки:

<span class="filename">Файл: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch15-smart-pointers/listing-15-20/src/lib.rs}}
```

<span class="caption">Листинг 15-20: Библиотека для отслеживания степени приближения того или иного значения к максимально допустимой величине и предупреждения, в случае если значение достигает определённого уровня</span>

Одна важная часть этого кода состоит в том, что типаж `Messenger` имеет один метод `send`, принимающий аргументами неизменяемую ссылку на `self` и текст сообщения. Он является интерфейсом, который должен иметь наш мок объект. Другой важной частью является то, что мы хотим проверить поведение метода `set_value` у типа `LimitTracker`. Мы можем изменить значение, которое передаём параметром `value`, но `set_value` ничего не возвращает и нет основания, чтобы мы могли бы проверить утверждения о выполнении метода. Мы хотим иметь возможность  сказать, что если мы создаём `LimitTracker` с чем-то, что реализует типаж `Messenger` и с определённым значением для `max`, то когда мы передаём разные числа в переменной `value` экземпляр self.messenger отправляет соответствующие сообщения.

Нам нужен мок объект, который вместо отправки электронного письма или текстового сообщения будет отслеживать сообщения, которые были ему поручены для отправки через `send`. Мы можем создать новый экземпляр мок объекта, создать `LimitTracker` с использованием мок объект для него, вызвать метод `set_value` у экземпляра `LimitTracker`, а затем проверить, что мок объект имеет ожидаемое сообщение. В листинге 15-21 показана попытка реализовать мок объект, чтобы сделать именно то что хотим, но анализатор заимствований не разрешит такой код:

<span class="filename">Файл: src/lib.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch15-smart-pointers/listing-15-21/src/lib.rs:here}}
```

<span class="caption">Листинг 15-21: Попытка реализовать <code>MockMessenger</code>, которая не была принята механизмом проверки заимствований</span>

Этот тестовый код определяет структуру `MockMessenger`, в которой есть поле `sent_messages` со значениями типа `Vec` из `String` для отслеживания сообщений, которые поручены структуре для отправки. Мы также определяем ассоциированную функцию `new`, чтобы было удобно создавать новые экземпляры `MockMessenger`, которые создаются с пустым списком сообщений. Затем мы реализуем типаж `Messenger` для типа `MockMessenger`, чтобы передать `MockMessenger` в `LimitTracker`. В сигнатуре метода `send` мы принимаем сообщение для передачи в качестве параметра и сохраняем его в `MockMessenger` внутри списка `sent_messages`.

В этом тесте мы проверяем, что происходит, когда `LimitTracker` сказано установить `value` в значение, превышающее 75 процентов от значения `max`. Сначала мы создаём новый `MockMessenger`, который будет иметь пустой список сообщений. Затем мы создаём новый `LimitTracker` и передаём ему ссылку на новый `MockMessenger` и `max` значение равное 100. Мы вызываем метод `set_value` у `LimitTracker` со значением 80, что составляет более 75 процентов от 100. Затем мы с помощью утверждения проверяем, что `MockMessenger` должен содержать одно сообщение из списка внутренних сообщений.

Однако с этим тестом есть одна проблема, показанная ниже:

```console
{{#include ../listings/ch15-smart-pointers/listing-15-21/output.txt}}
```

Мы не можем изменять `MockMessenger` для отслеживания сообщений, потому что метод `send` принимает неизменяемую ссылку на `self`. Мы также не можем принять предложение из текста ошибки, чтобы использовать `&mut self`, потому что тогда сигнатура `send` не будет соответствовать сигнатуре в определении типажа `Messenger` (не стесняйтесь попробовать и посмотреть, какое сообщение об ошибке получите вы).

Это ситуация, в которой внутренняя изменяемость может помочь! Мы сохраним `sent_messages` внутри типа `RefCell<T>`, а затем в методе `send` сообщение сможет изменить список `sent_messages` для хранения сообщений, которые мы видели. Листинг 15-22 показывает, как это выглядит:

<span class="filename">Файл: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch15-smart-pointers/listing-15-22/src/lib.rs:here}}
```

<span class="caption">Листинг 15-22: Использование <code>RefCell&lt;T&gt;</code> для изменения внутреннего значения, в то время как внешнее значение считается неизменяемым</span>

Поле `sent_messages` теперь имеет тип `RefCell<Vec<String>>` вместо `Vec<String>`. В функции `new` мы создаём новый экземпляр `RefCell<Vec<String>>` для пустого вектора.

Для реализации метода `send` первый параметр по-прежнему является неизменяемым для заимствования `self`, которое соответствует определению типажа. Мы вызываем `borrow_mut` для `RefCell<Vec<String>>` в `self.sent_messages`, чтобы получить изменяемую ссылку на значение внутри `RefCell<Vec<String>>`, которое является вектором. Затем мы можем вызвать `push` у изменяемой ссылки на вектор, чтобы отслеживать сообщения, отправленные во время теста.

Последнее изменение, которое мы должны сделать, заключается в утверждении для проверки: чтобы увидеть, сколько элементов находится во внутреннем векторе, мы вызываем метод `borrow` у `RefCell<Vec<String>>`, чтобы получить неизменяемую ссылку на внутренний вектор сообщений.

Теперь, когда вы увидели как использовать `RefCell<T>`, давайте изучим как он работает!

#### Отслеживание заимствований во время выполнения с помощью `RefCell<T>`

При создании неизменных и изменяемых ссылок мы используем синтаксис `&` и `&mut` соответственно. У типа `RefCell<T>`, мы используем методы `borrow` и `borrow_mut`, которые являются частью безопасного API, который принадлежит `RefCell<T>`. Метод `borrow` возвращает тип умного указателя `Ref<T>`, метод `borrow_mut` возвращает тип умного указателя `RefMut<T>`. Оба типа реализуют типаж `Deref`, поэтому мы можем рассматривать их как обычные ссылки.

Тип `RefCell<T>` отслеживает сколько умных указателей `Ref<T>` и `RefMut<T>` активны в данное время. Каждый раз, когда мы вызываем `borrow`, тип `RefCell<T>` увеличивает количество активных заимствований. Когда значение `Ref<T>` выходит из области видимости, то количество неизменяемых заимствований уменьшается на единицу. Как и с правилами заимствования во время компиляции, `RefCell<T>` позволяет иметь много неизменяемых заимствований или одно изменяемое заимствование в любой момент времени.

Если попытаться нарушить эти правила, то вместо получения ошибки компилятора, как это было бы со ссылками, реализация `RefCell<T>` будет вызывать панику во время выполнения. В листинге 15-23 показана модификация реализации `send` из листинга 15-22. Мы намеренно пытаемся создать два изменяемых заимствования активных для одной и той же области видимости, чтобы показать как `RefCell<T>` не позволяет нам делать так во время выполнения.

<span class="filename">Файл: src/lib.rs</span>

```rust,ignore,panics
{{#rustdoc_include ../listings/ch15-smart-pointers/listing-15-23/src/lib.rs:here}}
```

<span class="caption">Листинг 15-23: Создание двух изменяемых ссылок в одной области видимости, чтобы убедиться, что <code>RefCell&lt;T&gt;</code> вызовет панику</span>

Мы создаём переменную `one_borrow` для умного указателя `RefMut<T>` возвращаемого из метода `borrow_mut`. Затем мы создаём другое изменяемое заимствование таким же образом в переменной `two_borrow`. Это создаёт две изменяемые ссылки в одной области видимости, что недопустимо. Когда мы запускаем тесты для нашей библиотеки, код в листинге 15-23 компилируется без ошибок, но тест завершится неудачно:

```console
{{#include ../listings/ch15-smart-pointers/listing-15-23/output.txt}}
```

Обратите внимание, что код вызвал панику с сообщением `already borrowed: BorrowMutError`. Вот так тип `RefCell<T>` обрабатывает нарушения правил заимствования во время выполнения.

Решение отлавливать ошибки заимствования во время выполнения, а не во время компиляции, как мы сделали здесь, означает, что вы потенциально будете находить ошибки в своём коде на более поздних этапах разработки: возможно, не раньше, чем ваш код будет развернут в рабочем окружении. Кроме того, ваш код будет иметь небольшие потери производительности в процессе работы, поскольку заимствования будут отслеживаться во время выполнения, а не во время компиляции. Однако использование `RefCell<T>` позволяет написать объект-имитатор, который способен изменять себя, чтобы сохранять сведения о тех значениях, которые он получал, пока вы использовали его в контексте, где разрешены только неизменяемые значения. Вы можете использовать `RefCell<T>`, несмотря на его недостатки, чтобы получить больше функциональности, чем дают обычные ссылки.

### Наличие нескольких владельцев изменяемых данных путём объединения типов `Rc<T>` и `RefCell<T>`

Обычный способ использования `RefCell<T>` заключается в его сочетании с типом `Rc<T>`. Напомним, что тип `Rc<T>` позволяет иметь нескольких владельцев некоторых данных, но даёт только неизменяемый доступ к этим данным. Если у вас есть `Rc<T>`, который внутри содержит тип `RefCell<T>`, вы можете получить значение, которое может иметь несколько владельцев *и* которое можно изменять!

Например, вспомните пример cons списка листинга 15-18, где мы использовали `Rc<T>`, чтобы несколько списков могли совместно владеть другим списком. Поскольку `Rc<T>` содержит только неизменяемые значения, мы не можем изменить ни одно из значений в списке после того, как мы их создали. Давайте добавим тип `RefCell<T>`, чтобы получить возможность изменять значения в списках. В листинге 15-24 показано использование `RefCell<T>` в определении `Cons` так, что мы можем изменить значение хранящееся во всех списках:

<span class="filename">Файл: src/main.rs</span>

```rust
{{#rustdoc_include ../listings/ch15-smart-pointers/listing-15-24/src/main.rs}}
```

<span class="caption">Листинг 15-24: Использование <code>Rc&lt;RefCell&lt;i32&gt;&gt;</code> для создания <code>List</code>, который мы можем изменять</span>

Мы создаём значение, которое является экземпляром `Rc<RefCell<i32>>` и сохраняем его в переменной с именем `value`, чтобы получить к ней прямой доступ позже. Затем мы создаём `List` в переменной `a` с вариантом `Cons`, который содержит `value`. Нам нужно вызвать клонирование `value`, так как обе переменные `a` и `value` владеют внутренним значением `5`, а не передают владение из `value` в переменную `a` или не выполняют заимствование с помощью `a` переменной `value`.

Мы оборачиваем список у переменной `a` в тип `Rc<T>`, поэтому при создании списков в переменные `b` и `c` они оба могут ссылаться на `a`, что мы и сделали в листинге 15-18.

После создания списков `a`, `b` и `c` мы хотим добавить 10 к значению в `value`. Для этого вызовем `borrow_mut` у `value`, который использует функцию автоматического разыменования, о которой мы говорили в главе 5 (см. раздел ["Где находится оператор `->`?"]<!-- ignore -->) во внутреннее значение `RefCell<T>`. Метод `borrow_mut` возвращает умный указатель `RefMut<T>`, и мы используя оператор разыменования, изменяем внутреннее значение.

Когда мы печатаем `a`, `b` и `c` то видим, что все они имеют изменённое значение равное 15, а не 5:

```console
{{#include ../listings/ch15-smart-pointers/listing-15-24/output.txt}}
```

Эта техника довольно изящна! Используя `RefCell<T>`, мы получаем внешне неизменяемое значение `List`. Но мы можем использовать методы `RefCell<T>`, которые предоставляют доступ к его внутренностям, чтобы мы могли изменять наши данные, когда это необходимо. Проверка правил заимствования во время выполнения защищает нас от гонок данных, и иногда стоит немного пожертвовать производительностью ради такой гибкости наших структур данных. Обратите внимание, что `RefCell<T>` не работает для многопоточного кода! `Mutex<T>` - это thread-safe версия `RefCell<T>`, а `Mutex<T>` мы обсудим в главе 16.


["Где находится оператор `->`?"]: ch05-03-method-syntax.html#wheres-the---operator