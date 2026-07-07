# В чем принципиальная разница между Single-subscription и Broadcast стримами? В каких случаях стоит писать кастомный StreamTransformer?

> **Single-subscription** [Stream](stream_internals.md) допускает только одного слушателя и хранит события до подписки (ленивый — начинает генерацию при `listen()`). **Broadcast** Stream позволяет нескольким слушателям, но не буферизирует: кто не подписан — тот пропустил. Кастомный `StreamTransformer` оправдан, когда встроенных операторов (`map`, `where`, `expand`, `asyncMap`) недостаточно — например, нужно аккумулировать события с debounce, менять подписку (pause/resume), или преобразовывать single-subscription в broadcast с replay.

## Разбор

### Single-subscription Stream

Создаётся по умолчанию через `StreamController()` или `async*`. Контракт: **один `listen()`** за время жизни стрима.

```dart
// async* генератор — single-subscription
Stream<int> countDown(int from) async* {
  for (var i = from; i >= 0; i--) {
    await Future.delayed(const Duration(seconds: 1));
    yield i;
  }
}

final stream = countDown(5);
stream.listen(print);     // OK
// stream.listen(print);  // StateError: Stream already has subscriber
```

**Когда использовать:** чтение файлов, HTTP-ответ (chunked), WebSocket-канал «один к одному», результат Isolate.

### Broadcast Stream

Создаётся через `StreamController.broadcast()` или `.asBroadcastStream()`. Любое количество слушателей, но **без буферизации** — события уходят «в пустоту», если никто не подписан.

```dart
final controller = StreamController<String>.broadcast();

// Подписчик A
controller.stream.listen((e) => print('A: $e'));

controller.add('hello'); // A: hello

// Подписчик B подключился позже
controller.stream.listen((e) => print('B: $e'));

controller.add('world');
// A: world
// B: world
// B НЕ получил 'hello' — он опоздал
```

**Когда использовать:** UI-события, Supabase Realtime, глобальные шины событий, `ChangeNotifier`-like паттерны.

### Сводная таблица

| Свойство | Single-subscription | Broadcast |
|----------|---------------------|-----------|
| Слушатели | Один | Много |
| Буферизация до `listen()` | Да | Нет |
| Начало генерации | При `listen()` (ленивый) | Сразу / при `add()` |
| Повторный `listen()` | `StateError` | OK |
| Пример в Flutter | `HttpClientResponse` | `Stream` от `StreamController.broadcast()` |
| Supabase Realtime | — | `supabase.from('table').stream(...)` |

### Преобразование: asBroadcastStream()

```dart
final singleStream = File('log.txt').openRead(); // single-subscription

// Обёртка — теперь можно подписаться несколько раз
final broadcast = singleStream.asBroadcastStream();

broadcast.listen((chunk) => print('Logger: ${chunk.length} bytes'));
broadcast.listen((chunk) => analytics.track(chunk));
```

> **Осторожно:** `asBroadcastStream()` подписывается на оригинал немедленно. Если оригинальный стрим бесконечен (WebSocket), broadcast будет жить, пока вы явно не отмените подписку.

### Когда писать кастомный StreamTransformer

Встроенные операторы покрывают 90% задач:

```dart
stream
    .where((e) => e.isValid)         // фильтрация
    .map((e) => e.toEntity())        // трансформация
    .distinct()                       // дедупликация
    .asyncMap((e) => enrich(e))      // async трансформация
    .handleError((e) => log(e));     // обработка ошибок
```

Кастомный `StreamTransformer` нужен, когда:

1. **Stateful-логика:** нужно помнить предыдущие N событий (sliding window, buffer by count/time).
2. **Контроль подписки:** нужно управлять `pause`/`resume`/`cancel` нестандартно.
3. **Изменение кардинальности:** одно входное событие → несколько выходных (или наоборот).

```dart
/// Debounce: пропускает событие, только если после него прошло [duration] тишины.
class DebounceTransformer<T> extends StreamTransformerBase<T, T> {
  DebounceTransformer(this.duration);
  final Duration duration;

  @override
  Stream<T> bind(Stream<T> stream) {
    Timer? timer;
    late final StreamController<T> controller;

    controller = StreamController<T>(
      onListen: () {
        final subscription = stream.listen(
          (data) {
            timer?.cancel();
            timer = Timer(duration, () => controller.add(data));
          },
          onError: controller.addError,
          onDone: () {
            timer?.cancel();
            controller.close();
          },
        );
        controller.onCancel = () {
          timer?.cancel();
          subscription.cancel();
        };
      },
    );

    return controller.stream;
  }
}

// Использование
searchField.stream
    .transform(DebounceTransformer(const Duration(milliseconds: 300)))
    .listen((query) => bloc.add(SearchRequested(query)));
```

### Практические паттерны во Flutter

```dart
// Supabase Realtime — broadcast stream в репозитории
Stream<List<Incident>> watchIncidents() {
  return supabase
      .from('incidents')
      .stream(primaryKey: ['id'])    // broadcast
      .map((rows) => rows.map(IncidentDto.fromJson).map((d) => d.toEntity()).toList());
}

// BLoC подписывается на стрим
class IncidentListBloc extends Bloc<IncidentEvent, IncidentState> {
  IncidentListBloc(this._repo) : super(const IncidentInitial()) {
    on<IncidentWatchStarted>((event, emit) async {
      await emit.forEach(
        _repo.watchIncidents(),
        onData: (incidents) => IncidentLoaded(incidents),
        onError: (e, st) => IncidentError(ServerFailure(e.toString())),
      );
    });
  }
  final IncidentRepository _repo;
}
```

## Что почитать

* [Dart: Creating streams](https://dart.dev/articles/creating-streams)
* [Dart: StreamTransformer class](https://api.dart.dev/stable/dart-async/StreamTransformer-class.html)
* [Dart: Single vs Broadcast streams](https://dart.dev/tutorials/language/streams#two-kinds-of-streams)
