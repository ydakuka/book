## Чтение файла

Теперь добавим возможность чтения файла, указанного как аргумент командной строки `file_path`. Во-первых, нам нужен пример файла для тестирования: мы будем использовать файл с небольшим объёмом текста в несколько строк с несколькими повторяющимися словами. В листинге 12-3 представлено стихотворение Эмили Дикинсон, которое будет хорошо работать! Создайте файл с именем *poem.txt* в корне вашего проекта и введите стихотворение "I’m nobody! Who are you?"

<span class="filename">Файл: poem.txt</span>

```text
{{#include ../listings/ch12-an-io-project/listing-12-03/poem.txt}}
```

<span class="caption">Листинг 12-3: Стихотворение Эмили Дикинсон - хороший пример для проверки</span>

Текст на месте, отредактируйте *src/main.rs* и добавьте код для чтения файла, как показано в листинге 12-4.

<span class="filename">Файл: src/main.rs</span>

```rust,should_panic,noplayground
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-04/src/main.rs:here}}
```

<span class="caption">Листинг 12-4: Чтение содержимого файла указанного во втором аргументе</span>

Во-первых, мы добавляем ещё одно объявление `use` чтобы подключить соответствующую часть стандартной библиотеки: нам нужен `std::fs` для обработки файлов.

В `main` мы добавили новый оператор: функция `fs::read_to_string` принимает `file_path`, открывает этот файл и возвращает содержимое файла как `std::io::Result<String>`.

После этого выражения мы снова добавили временный вывод `println!` для печати значения `contents` после чтения файла, таким образом мы можем проверить, что программа отрабатывает до этого места.

Давайте запустим этот код с любой строкой в качестве первого аргумента командной строки (потому что мы ещё не реализовали поисковую часть) и файл *poem.txt* как второй аргумент:

```console
{{#rustdoc_include ../listings/ch12-an-io-project/listing-12-04/output.txt}}
```

Отлично! Этот код прочитал и затем напечатал содержимое файла. Но у программы есть несколько недостатков. Прежде всего, функция `main` решает слишком много задач: как правило функция понятнее и проще в обслуживании если она воплощает только одну идею. Другая проблема заключается в том, что мы не обрабатываем ошибки так хорошо, как могли бы. Пока наша программа небольшая, то эти недостатки не являются большой проблемой, но по мере роста программы эти недостатки будет всё труднее исправлять. Хорошей практикой является начинать рефакторинг на ранней стадии разработки программы, потому что гораздо проще рефакторить меньшие объёмы кода. Мы сделаем это далее.
