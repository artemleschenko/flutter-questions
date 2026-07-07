# flutter-questions

Конспекты к типовым вопросам на собеседованиях (Flutter / Dart). Формат ответа: краткий ответ → разбор → ссылки.

## 01 — Computer Science

* [В чем разница между декларативными и императивным подходами написаниями кода?](01_cs/imp_vs_dec.md)
* [Зачем нужна нотация Big-O? Всегда ли она показывает реальную скорость работы алгоритма?](01_cs/big_o.md)
* [В чем разница между асинхронностью и многопоточностью?](01_cs/multithreading_vs_async.md)
* [В чем принципиальная разница между стеком (Stack) и очередью (Queue)? Приведи примеры, где их логично использовать в реальном приложении.](01_cs/queue_vs_stack.md)
* [Расскажи структуру OSI. За что отвечает каждый из слоев и какие протоколы представлены на них?](01_cs/osi_levels.md)
* [Какие есть механизмы синхронизации потоков? В чем преимущества и недостатки каждого метода.](01_cs/sync_mechanisms.md)
* [Что такое Snowflake ID? Сравни с UUID](01_cs/snowflake_id.md)

## 02 — Dart Core

* [В чем разница между var, final и const? В каких случаях ты выберешь каждый из них?](02_dart_core/var_final_and_const.md)
* [Что такое Sound Null Safety? Какие операторы для работы с nullable-типами ты знаешь? Почему использование `!` считается плохой практикой?](02_dart_core/sound_null_safety.md)
* [Для чего нужно ключевое слово late? Какие риски связаны с его использованием?](02_dart_core/keyword_late.md)
* [В чем разница между оператором == и идентичностью identical()? Зачем нужен пакет equatable и как он работает внутри?](02_dart_core/identical_and_compare.md)
* [В чем разница между List, Set и Map? В каком случае ты используешь Set вместо List?](02_dart_core/list_set_and_map.md)
* [Что такое Generics? Расскажи про концепцию Soundness в системе типов Dart.](02_dart_core/generics.md)
* [Что понимают под F-bounds в Dart? В каких случаях при проектировании базовых классов/архитектуры тебе понадобится использовать рекурсивные дженерики?](02_dart_core/f_bounds.md)
* [Что означает ключевое слово covariant в Dart и почему иногда оно необходимо при переопределении методов?](02_dart_core/covariant.md)
* [Что такое Records в Dart 3 и как они позволяют избавиться от создания лишних классов-моделей?](02_dart_core/records.md)
* [Что такое Tear-offs и как передача функции как объекта влияет на производительность и читаемость кода?](02_dart_core/tear-offs.md)

## 03 — OOP

* [Расскажи принципы ООП. Как знание этих принципов помогает тебе в работе?](03_oop/oop_principles.md)
* [Объясни своими словами, чем отличается абстрактный класс от обычного интерфейса в реалиях Dart (учитывая, что ключевого слова interface до Dart 3 не было)?](03_oop/abstract_vs_interface.md)
* [Чем отличаются расширение и имплементация классов?](03_oop/extends_vs_implements.md)
* [В чем разница между mixin и обычным наследованием? Какие ограничения есть у миксинов в Dart?](03_oop/mixin_vs_inheritance.md)
* [В чем разница между модификаторами base, interface, final и sealed при объявлении классов?](03_oop/class_modifiers.md)

## 04 — Async & Runtime

* [Что такое Future? В чем разница между async/await и использованием метода .then()?](04_async_runtime/future_async_await.md)
* [Как в Dart перехватывать и обрабатывать ошибки при работе с Future?](04_async_runtime/future_error_handling.md)
* [Как работает Event Loop? Как приоритетность очереди микрозадач влияет на отзывчивость интерфейса?](04_async_runtime/event_loop.md)
* [В чем принципиальная разница между Single-subscription и Broadcast стримами?](04_async_runtime/streams_types.md)
* [Как работает сборщик мусора (Garbage Collector)? Чем поколения в Dart GC отличаются от Java?](04_async_runtime/garbage_collector.md)
* [Почему циклические ссылки не приводят к утечкам памяти в Dart?](04_async_runtime/cyclic_references.md)
* [Когда объект должен удерживаться через WeakReference, а не через обычную ссылку?](04_async_runtime/weak_reference.md)
* [Для каких задач предназначены Finalizer и почему их нельзя использовать как замену dispose()?](04_async_runtime/finalizer.md)
* [Каков жизненный цикл объекта в Dart с момента выделения памяти в Young Space до его удаления?](04_async_runtime/object_lifecycle.md)
* [Как работает Stream под капотом?](04_async_runtime/stream_internals.md)
* [Что такое Zone в Dart и как этот механизм используется для перехвата необработанных ошибок?](04_async_runtime/zones.md)

## 06 — Flutter UI

* [Расскажи как реализовать Accessibility во Flutter приложениях](06_flutter_ui/accessibility.md)

## 07 — Under the Hood

* [Объясни связь между Widget Tree, Element Tree и RenderObject Tree. Зачем Flutter использует именно три дерева?](07_under_the_hood/trees_relationship.md)
* [Как работают Extension Methods под капотом? Можно ли добавить новое поле состояния внутрь класса через Extension?](07_under_the_hood/extension_methods.md)
* [Что это за виджет RepaintBoundary и когда его использование может спасти FPS?](07_under_the_hood/repaint_boundary.md)
* [Почему виджеты IntrinsicHeight и IntrinsicWidth считаются «дорогими»?](07_under_the_hood/intrinsic_widgets.md)

## 10 — Testing & Tools

* [Что такое flavors? Как ты организуешь работу с ними?](10_testing_and_tools/flavors.md)

---

Шаблон для новых ответов: [TEMPLATE.md](TEMPLATE.md)
