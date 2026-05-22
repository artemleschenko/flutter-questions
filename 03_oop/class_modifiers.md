# В чем разница между модификаторами base, interface, final и sealed при объявлении классов?

> Модификаторы Dart 3 задают **кто и как может подтипировать класс вне вашей библиотеки**: `base` — только `extends` (наследовать реализацию), не `implements`; `interface` — только `implements`, не `extends`; `final` — ни то ни другое снаружи; `sealed` — закрытая иерархия в одном файле/библиотеке для исчерпывающего `switch`. Они решают проблему хрупкого базового класса в публичных API (в т.ч. пакетов Flutter). `abstract` по-прежнему запрещает прямое создание экземпляра, но не заменяет эти ограничения.

## Разбор

До Dart 3 любой публичный класс можно было и **расширить**, и **реализовать** из другого пакета — автор SDK не мог жёстко зафиксировать намерение. Модификаторы делают контракт явным для потребителей и для анализатора.

### Сводная таблица (вне своей library)

| Модификатор | Создать `new`? | `extends`? | `implements`? |
|-------------|----------------|------------|-------------|
| *(нет)* `class` | Да | Да | Да |
| `abstract class` | Нет* | Да | Да |
| `base class` | Да | Да | **Нет** |
| `interface class` | Да | **Нет** | Да |
| `final class` | Да | **Нет** | **Нет** |
| `sealed class` | Нет** | **Нет*** | **Нет*** |

\* Можно через factory внутри иерархии.  
\** Неявно abstract.  
\*** Подтипы только в **той же** library.

### `base` — «наследуй мою реализацию, не подделывай»

Гарантия: если вы добавите метод с телом в `base class`, все **наследники** получат его — никто снаружи не «обманул» через `implements` пустышкой.

```dart
// lib/network_client.dart
base class NetworkClient {
  void send(Uri url) { /* общая логика */ }
}

// lib/my_app_client.dart
base class LoggingClient extends NetworkClient { /* OK */ }

// ERROR: implements NetworkClient в другой library
class FakeClient implements NetworkClient { /* запрещено */ }
```

Подтип, который **extends** `base`, тоже должен быть помечен `base`, `final` или `sealed`.

*Flutter-контекст:* внутренние классы SDK, где важно, чтобы наследники не ломали инварианты `RenderObject`, часто проектируют с ограничениями; в своих пакетах `base` защищает от «полу-моков» через `implements`.

### `interface` — «подключайся через контракт, не через extends»

Снаружи можно только `implements`. Методы класса, вызывающие друг друга через `this`, не попадут на «неожиданный» override из наследника в другом пакете.

```dart
// lib/location_service.dart
interface class LocationService {
  Future<Coordinates> current();
}

class GpsLocation implements LocationService {
  @override
  Future<Coordinates> current() async { /* ... */ }
}

// ERROR: class GpsLocation extends LocationService
```

**`abstract interface class`** — идиоматический чистый интерфейс Dart 3 (нельзя инстанцировать, нельзя extend снаружи).

### `final` — «тип закрыт для подтипизации снаружи»

Нельзя ни наследовать, ни реализовать из другой library. Безопасно расширять API класса без риска, что кто-то переопределил метод и сломал внутренние вызовы.

```dart
final class AppConfig {
  AppConfig({required this.apiUrl});
  final String apiUrl;
}
```

В **той же** library `final` можно расширять/имплементировать — для тестов внутри пакета.

*Когда выбрать вместо `sealed`:* хотите закрыть иерархию, но **не** обещаете исчерпывающий список подтипов (можете добавить подкласс в том же файле позже без правки всех `switch` по проекту).

### `sealed` — «известный набор вариантов»

Подтипы только в одной library; сам `sealed` не инстанцируется (implicit abstract). Главная выгода — **exhaustive switch** в Dart 3:

```dart
sealed class LoadState {}

class LoadIdle extends LoadState {}
class LoadInProgress extends LoadState {}
class LoadSuccess extends LoadState {
  LoadSuccess(this.data);
  final String data;
}
class LoadFailure extends LoadState {
  LoadFailure(this.error);
  final Object error;
}

Widget buildBody(LoadState state) {
  return switch (state) {
    LoadIdle() => const Text('Tap to load'),
    LoadInProgress() => const CircularProgressIndicator(),
    LoadSuccess(:final data) => Text(data),
    LoadFailure(:final error) => Text('$error'),
  }; // анализатор проверит полноту веток
}
```

Идеально для **UI-состояний**, событий FSM, AST, result-типов (`Success | Failure`).

*Отличие от `final`:* `sealed` **фиксирует** варианты для компилятора; `final` только запрещает внешнее подтипирование.

### `abstract` — не путать с остальными

`abstract` говорит: «экземпляр этого класса не создаём», но **сам по себе** не запрещает `extends`/`implements` снаружи. Частые комбинации:

* `abstract base class` — нельзя создать, наследовать реализацию можно, имплементировать снаружи нельзя.
* `abstract interface class` — чистый контракт.
* `abstract` + `sealed` — избыточно: `sealed` уже implicit abstract.

### Комбинации и запреты

Порядок в объявлении: `(abstract)?` → `(base|interface|final|sealed)?` → `(mixin)?` → `class`.

Нельзя вместе:

* `abstract` + `sealed` (sealed уже abstract),
* `interface` / `final` / `sealed` + `mixin` в одной декларации.

### Практика для Flutter-разработчика

| Задача | Модификатор |
|--------|-------------|
| Состояния экрана / union-типы | `sealed class` + `switch` |
| Публичный API пакета без наследников | `final class` |
| Контракт репозитория | `abstract interface class` |
| База с общей логикой, запрет «пустых» implements | `base class` |

```dart
// domain/result.dart
sealed class Result<T> {
  const Result();
}

final class Ok<T> extends Result<T> {
  const Ok(this.value);
  final T value;
}

final class Err<T> extends Result<T> {
  const Err(this.message);
  final String message;
}
```

Здесь `sealed` фиксирует варианты, `final` на кейсах не даёт плодить подтипы `Ok`/`Err` снаружи.

## Что почитать

* [Dart: Class modifiers](https://dart.dev/language/class-modifiers)
* [Dart: Class modifiers for API maintainers](https://dart.dev/language/class-modifiers-for-apis)
* [Dart: Modifier reference](https://dart.dev/language/modifier-reference)
* [QuickBird: Class Modifiers in Dart](https://quickbirdstudios.com/blog/flutter-dart-class-modifiers/)
