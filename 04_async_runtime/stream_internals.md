# Как работает Stream под капотом?

> `Stream<T>` — абстракция над **асинхронной последовательностью** событий. Под капотом Stream реализован через цепочку `StreamController → StreamSubscription → EventSink`. Когда вы вызываете `listen()`, создаётся `StreamSubscription`, которая регистрирует callbacks в контроллере. При `add(event)` контроллер помещает событие в [микрозадачу](event_loop.md), которая доставляет его подписчикам через [Event Loop](event_loop.md). Stream поддерживает back-pressure через `pause()`/`resume()` и может быть [single-subscription или broadcast](streams_types.md).

## Разбор

### Архитектура изнутри

```
                    ┌───────────────────────────────────────────┐
                    │              Источник событий              │
                    │  (I/O, Timer, Supabase, пользовательский) │
                    └──────────────────┬────────────────────────┘
                                       │ add(event) / addError(e)
                                       ▼
                    ┌───────────────────────────────────────────┐
                    │          StreamController<T>               │
                    │  ┌─────────────────────────────────────┐  │
                    │  │ EventSink (onData, onError, onDone) │  │
                    │  │ _state: _SUBSCRIPTION_MASK           │  │
                    │  │ _pendingEvents: Queue?               │  │
                    │  └─────────────────────────────────────┘  │
                    └──────────────────┬────────────────────────┘
                                       │ .stream
                                       ▼
                    ┌───────────────────────────────────────────┐
                    │              Stream<T>                     │
                    │  (фасад, единственный публичный API)      │
                    └──────────────────┬────────────────────────┘
                                       │ .listen(onData, onError, onDone)
                                       ▼
                    ┌───────────────────────────────────────────┐
                    │          StreamSubscription<T>             │
                    │  ┌─────────────────────────────────────┐  │
                    │  │ _onData: void Function(T)?          │  │
                    │  │ _onError: Function?                  │  │
                    │  │ _onDone: void Function()?            │  │
                    │  │ _isPaused: bool                      │  │
                    │  └─────────────────────────────────────┘  │
                    └───────────────────────────────────────────┘
```

### Что происходит при listen()

```dart
final controller = StreamController<int>();
final stream = controller.stream;

// Внутри listen():
// 1. Создаётся StreamSubscription с вашими callbacks
// 2. Subscription регистрируется в Controller
// 3. Если есть onListen callback у контроллера — вызывается
// 4. Если были буферизованные события — планируются как микрозадачи
final sub = stream.listen(
  (data) => print('data: $data'),    // _onData
  onError: (e) => print('error: $e'), // _onError
  onDone: () => print('done'),        // _onDone
);
```

### Что происходит при add()

```dart
controller.add(42);
// Внутри:
// 1. Проверяется: есть ли подписчик?
//    - Нет подписчика (single-sub) → буферизация в _pendingEvents
//    - Нет подписчика (broadcast) → событие отброшено
// 2. Подписчик на паузе? → буферизация
// 3. Подписчик активен → scheduleMicrotask(() => subscription._onData(42))
```

**Важно:** события доставляются **асинхронно** через микрозадачи, даже если `add()` вызван синхронно. Это гарантирует, что `listen` всегда получает события после возврата из текущего синхронного кода.

### Back-pressure: pause / resume

```dart
final sub = stream.listen((chunk) async {
  sub.pause();  // ← Говорим источнику: «подожди, я занят»
  await processChunk(chunk);  // тяжёлая обработка
  sub.resume(); // ← «Готов к следующему»
});

// Контроллер знает о pause через onPause/onResume:
final controller = StreamController<Chunk>(
  onPause: () => socket.pause(),   // Прекращаем чтение из сокета
  onResume: () => socket.resume(), // Возобновляем чтение
);
```

Это позволяет **не терять данные** при медленном потребителе — в отличие от broadcast, который не буферизирует.

### async* — синтаксический сахар для StreamController

```dart
// Это:
Stream<int> numbers() async* {
  for (var i = 0; i < 5; i++) {
    await Future.delayed(Duration(seconds: 1));
    yield i;
  }
}

// Компилятор превращает примерно в:
Stream<int> numbers() {
  late StreamController<int> controller;
  controller = StreamController<int>(
    onListen: () async {
      for (var i = 0; i < 5; i++) {
        await Future.delayed(Duration(seconds: 1));
        if (controller.isClosed) return;
        controller.add(i);           // yield → add
        // yield также неявно проверяет pause/resume
      }
      controller.close();
    },
  );
  return controller.stream;
}
```

`yield*` делегирует весь Stream:

```dart
Stream<int> combined() async* {
  yield* numbers();      // все события из numbers()
  yield* moreNumbers();  // затем все из moreNumbers()
}
```

### Цепочка трансформаций

Операторы (`map`, `where`, `expand`) создают новые Stream'ы, каждый с собственной подпиской:

```dart
stream
  .where((x) => x > 0)       // Новый Stream + Subscription
  .map((x) => x * 2)         // Ещё один Stream + Subscription
  .listen(print);             // Финальная Subscription
```

```
Оригинальный Stream
    │ listen()
    ▼
WhereStream (фильтрация)
    │ listen()
    ▼
MapStream (трансформация)
    │ listen()
    ▼
Ваш callback (print)
```

Каждое звено — обёртка над предыдущим. Событие проходит цепочку **синхронно** внутри одной микрозадачи.

### StreamController: sync vs async

```dart
// По умолчанию — async: события через микрозадачу
final asyncCtrl = StreamController<int>();

// sync: событие доставляется немедленно в том же стеке вызовов
final syncCtrl = StreamController<int>.broadcast(sync: true);
```

> **Осторожно с sync:** если listener вызывает `add()` внутри своего callback → бесконечная рекурсия. Используйте sync только когда понимаете последствия (например, внутри framework кода).

### Жизненный цикл Stream в Flutter-виджете

```dart
class _IncidentListState extends State<IncidentList> {
  late final StreamSubscription<List<Incident>> _sub;

  @override
  void initState() {
    super.initState();
    _sub = widget.repo
        .watchIncidents()          // Stream из Supabase Realtime
        .listen(
          (incidents) => setState(() => _items = incidents),
          onError: (e) => setState(() => _error = e),
        );
  }

  @override
  void dispose() {
    _sub.cancel();  // ✅ ОБЯЗАТЕЛЬНО — иначе утечка
    super.dispose();
  }
}
```

При `cancel()`:
1. Subscription отписывается от Controller.
2. Controller вызывает `onCancel` callback.
3. Если это последний подписчик single-subscription — Controller может закрыться.

## Что почитать

* [Dart: Creating streams](https://dart.dev/articles/creating-streams)
* [Dart: Stream class API](https://api.dart.dev/stable/dart-async/Stream-class.html)
* [Dart: StreamController API](https://api.dart.dev/stable/dart-async/StreamController-class.html)
* [Dart: Asynchronous programming — streams](https://dart.dev/tutorials/language/streams)
