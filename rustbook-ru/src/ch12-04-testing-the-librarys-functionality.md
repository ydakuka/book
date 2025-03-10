## Развитие функциональности библиотеки разработкой на основе тестов

Теперь, когда мы извлекли логику в *src/lib.rs* и оставили разбор аргументов командной строки и обработку ошибок в *src/main.rs*, стало гораздо проще писать тесты для основной функциональности нашего кода. Мы можем вызывать функции напрямую с различными аргументами и проверить возвращаемые значения без необходимости вызова нашего двоичного файла из командной строки.

В этом разделе в программу `minigrep` мы добавим логику поиска с использованием процесса разработки через тестирование (TDD), который следует этим шагам:

1. Напишите тест, который не прошёл и запустите его, чтобы убедиться, что он не прошёл по той причине, которую вы ожидаете.
2. Пишите или изменяйте ровно столько кода, чтобы успешно выполнился новый тест.
3. Выполните рефакторинг кода, который вы только что добавили или изменили, и убедитесь, что тесты продолжают проходить.
4. Повторите с шага 1!

Хотя это всего лишь один из многих способов написания программного обеспечения, TDD может помочь в разработке кода. Написание теста перед написанием кода, обеспечивающего прохождение теста, помогает поддерживать высокое покрытие тестами на протяжении всего процесса разработки.

Мы протестируем реализацию функциональности, которая делает поиск строки запроса в содержимом файла и создание списка строк, соответствующих запросу. Мы добавим эту функциональность в функцию под названием `search` .

### Написание теста с ошибкой

Поскольку они нам больше не нужны, давайте удалим строки с `println!`, которые мы использовали для проверки поведения программы в *src/lib.rs* и *src/main.rs*. Затем в *src/lib.rs* мы добавим модуль `tests` с тестовой функцией, как делали это в [главе 11.]<!-- ignore --> Тестовая функция определяет поведение, которое мы хотим проверить в функции <code>search</code>: она должна принимать запрос и текст для поиска, а возвращать только те строки из текста, которые содержат запрос. В листинге 12-15 показан этот тест, который пока не компилируется.

<span class="filename">Файл: src/lib.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-15/src/lib.rs:here}}
```

<span class="caption">Листинг 12-15. Создание теста с ошибкой для функции <code>search</code>, которую мы хотели бы получить</span>

Этот тест ищет строку `"duct"`. Текст, который мы ищем состоит из трёх строк, только одна из которых содержит `"duct"` (обратите внимание, что обратная косая черта после открывающей двойной кавычки говорит Rust не помещать символ новой строки в начало содержимого этого строкового литерала). Мы проверяем, что значение, возвращаемое функцией `search`, содержит только ожидаемую нами строку.

Мы не можем запустить этот тест и увидеть сбой, потому что тест даже не компилируется: функции `search` ещё не существует! В соответствии с принципами TDD мы добавим ровно столько кода, чтобы тест компилировался и запускался, добавив определение функции `search`, которая всегда возвращает пустой вектор, как показано в листинге 12-16. Потом тест должен скомпилироваться и потерпеть неудачу при запуске, потому что пустой вектор не равен вектору, содержащему строку `"safe, fast, productive."`

<span class="filename">Файл: src/lib.rs</span>

```rust,noplayground
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-16/src/lib.rs:here}}
```

<span class="caption">Листинг 12-16. Определение функции <code>search</code>, достаточное, чтобы тест скомпилировался</span>

Заметьте, что в сигнатуре `search` нужно явно указать время жизни `'a` для аргумента `contents` и возвращаемого значения. Напомним из [Главы 10]<!-- ignore -->, что параметры времени жизни указывают с временем жизни какого аргумента связано время жизни возвращаемого значения. В данном случае мы говорим, что возвращаемый вектор должен содержать срезы строк, ссылающиеся на содержимое аргумента `contents` (а не аргумента `query`).

Другими словами, мы говорим Rust, что данные, возвращаемые функцией `search`, будут жить до тех пор, пока живут данные, переданные в функцию `search` через аргумент `contents`. Это важно! Чтобы ссылки были действительными, данные, на которые ссылаются *с помощью* срезов тоже должны быть действительными; если компилятор предполагает, что мы делаем строковые срезы переменной `query`, а не переменной `contents`, он неправильно выполнит проверку безопасности.

Если мы забудем аннотации времени жизни и попробуем скомпилировать эту функцию, то получим следующую ошибку:

```console
{{#include ../listings/ch12-an-io-project/output-only-02-missing-lifetimes/output.txt}}
```

Rust не может понять, какой из двух аргументов нам нужен, поэтому нужно сказать ему об этом. Так как `contents` является тем аргументом, который содержит весь наш текст, и мы хотим вернуть части этого текста, которые совпали при поиске, мы понимаем, что `contents` является аргументом, который должен быть связан с возвращаемым значением временем жизни.

Другие языки программирования не требуют от вас связывания в сигнатуре аргументов с возвращаемыми значениями, но после определённой практики вам станет проще. Можете сравнить этот пример с разделом [«Проверка ссылок с временами жизни»](ch10-03-lifetime-syntax.html#validating-references-with-lifetimes)<!-- ignore --> главы 10.

Запустим тест:

```console
{{#include ../listings/ch12-an-io-project/listing-12-16/output.txt}}
```

Отлично. Наш тест не сработал, как мы и ожидали. Давайте сделаем так, чтобы он срабатывал!

### Написание кода для прохождения теста

Сейчас наш тест не проходит, потому что мы всегда возвращаем пустой вектор. Чтобы исправить это и реализовать `search`, наша программа должна выполнить следующие шаги:

- Итерироваться по каждой строке содержимого.
- Проверить, содержит ли данная строка искомую.
- Если это так, добавить её в список значений, которые мы возвращаем.
- Если это не так, ничего не делать.
- Вернуть список результатов.

Давайте проработаем каждый шаг, начиная с перебора строк.

#### Перебор строк с помощью метода `lines`

В Rust есть полезный метод для построчной итерации строк, удобно названый `lines`, как показано в листинге 12-17. Обратите внимание, код пока не компилируется.

<span class="filename">Файл: src/lib.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-17/src/lib.rs:here}}
```

<span class="caption">Листинг 12-17: Итерация по каждой строке из <code>contents</code></span>

Метод `lines` возвращает итератор. Мы подробно поговорим об итераторах в [Главе 13]<!-- ignore -->, но вспомните, что вы видели этот способ использования итератора в [Листинге 3-5]<!-- ignore -->, где мы использовали цикл `for` с итератором, чтобы выполнить некоторый код для каждого элемента в коллекции.

#### Поиск в каждой строке текста запроса

Далее мы проверяем, содержит ли текущая строка нашу искомую строку. К счастью, у строк есть полезный метод `contains`, который именно это и делает! Добавьте вызов метода `contains` в функции `search`, как показано в листинге 12-18. Обратите внимание, что это все ещё не компилируется.

<span class="filename">Файл: src/lib.rs</span>

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-18/src/lib.rs:here}}
```

<span class="caption">Листинг 12-18. Добавление проверки, содержится ли <code>query</code> в строке</span>

На данный момент мы наращиваем функциональность. Чтобы заставить это скомпилироваться, нам нужно вернуть значение из тела функции, как мы указали в сигнатуре функции.

#### Сохранение совпавшей строки

Чтобы завершить эту функцию, нам нужен способ сохранить совпадающие строки, которые мы хотим вернуть. Для этого мы можем создать изменяемый вектор перед циклом `for` и вызывать метод `push` для сохранения `line` в векторе. После цикла `for` мы возвращаем вектор, как показано в листинге 12-19.

<span class="filename">Файл: src/lib.rs</span>

```rust,ignore
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-19/src/lib.rs:here}}
```

<span class="caption">Листинг 12-19. Сохранение совпадающих строк, чтобы вернуть их</span>

Теперь функция `search` должна возвратить только строки, содержащие `query`, и тест должен пройти. Запустим его:

```console
{{#include ../listings/ch12-an-io-project/listing-12-19/output.txt}}
```

Наш тест пройден, значит он работает!

На этом этапе мы могли бы рассмотреть возможности изменения реализации функции поиска, сохраняя прохождение тестов и поддерживая имеющуюся функциональность. Код в функции поиска не так уж плох, но он не использует некоторые полезные функции итераторов. Вернёмся к этому примеру в [главе 13](ch13-02-iterators.html)<!-- ignore -->, где будем исследовать итераторы подробно, и посмотрим как его улучшить.

#### Использование функции `search` в функции `run`

Теперь, когда функция `search` работает и протестирована, нужно вызвать `search` из нашей функции `run`. Нам нужно передать значение `config.query` и `contents`, которые `run` читает из файла, в функцию `search`. Тогда `run` напечатает каждую строку, возвращаемую из `search`:

<span class="filename">Файл: src/lib.rs</span>

```rust,ignore
{{#rustdoc_include ../listings/ch12-an-io-project/no-listing-02-using-search-in-run/src/lib.rs:here}}
```

Мы по-прежнему используем цикл `for` для возврата каждой строки из функции `search` и её печати.

Теперь вся программа должна работать! Давайте попробуем сначала запустить её со словом «frog», которое должно вернуть только одну строчку из стихотворения Эмили Дикинсон:

```console
{{#include ../listings/ch12-an-io-project/no-listing-02-using-search-in-run/output.txt}}
```

Здорово! Теперь давайте попробуем слово, которое будет соответствовать нескольким строкам, например «body»:

```console
{{#include ../listings/ch12-an-io-project/output-only-03-multiple-matches/output.txt}}
```

И наконец, давайте удостоверимся, что мы не получаем никаких строк, когда ищем слово, отсутствующее в стихотворении, например «monomorphization»:

```console
{{#include ../listings/ch12-an-io-project/output-only-04-no-matches/output.txt}}
```

Отлично! Мы создали собственную мини-версию классического инструмента и научились тому, как структурировать приложения. Мы также немного узнали о файловом вводе и выводе, временах жизни, тестировании и разборе аргументов командной строки.

Чтобы завершить этот проект, мы кратко продемонстрируем пару вещей: как работать с переменными окружения и как печатать в стандартный поток ошибок, обе из которых полезны при написании консольных программ.


[главе 11.]: ch11-01-writing-tests.html#the-anatomy-of-a-test-function
[Главы 10]: ch10-03-lifetime-syntax.html
[Листинге 3-5]: ch03-05-control-flow.html#looping-through-a-collection-with-for
[Главе 13]: ch13-02-iterators.html