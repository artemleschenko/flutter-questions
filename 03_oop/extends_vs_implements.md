# Чем отличаются расширение и имплементация классов?

> **`extends` (расширение)** — «я *являюсь* подтипом и наследую реализацию»: один суперкласс, доступ к `super`, переопределение методов. **`implements` (имплементация)** — «я *поддерживаю* тот же публичный API, но пишу (или получаю) реализацию сам»: можно несколько интерфейсов, унаследованного кода от типа нет. В Dart разница критична для тестов, моков и архитектуры Flutter: репозитории и сервисы чаще `implements`, виджеты и render-объекты — `extends`.

## Разбор

Dart — язык с **одиночным наследованием реализации** и **множественной имплементацией интерфейсов**. Миксины (`with`) — третий механизм, но про них отдельный файл.

### Расширение (`extends`)

Подкласс получает поля, методы и конструкторную цепочку суперкласса (если они доступны). Можно дополнять и переопределять через `@override`, вызывать родителя через `super`.

```dart
class Animal {
  void breathe() => print('inhale/exhale');
}

class Dog extends Animal {
  void bark() => print('woof');

  @override
  void breathe() {
    super.breathe();
    print('panting');
  }
}
```

*   *Аналогия:* вы не просто умеете то же, что родитель — вы **часть той же линии** (собака *является* животным).
*   *Во Flutter:* `class MyApp extends StatelessWidget` — вы наследуете всю инфраструктуру виджета (`build`, `key`, `createElement`).

### Имплементация (`implements`)

Класс обязан предоставить **все публичные instance-члены** интерфейса (включая getters/setters/operators), но **не копирует** код суперкласса. Приватные члены интерфейса не видны снаружи библиотеки — их тоже нужно воспроизвести, если вы в той же library.

```dart
abstract class Storage {
  Future<void> write(String key, String value);
  Future<String?> read(String key);
}

class MemoryStorage implements Storage {
  final _data = <String, String>{};

  @override
  Future<void> write(String key, String value) async {
    _data[key] = value;
  }

  @override
  Future<String?> read(String key) async => _data[key];
}
```

*   *Аналогия:* вы сдаёте экзамен по **программе курса** (интерфейсу), но конспект пишете сами — не копируете чужую тетрадь.
*   *Во Flutter:* `class FakePathProvider implements PathProviderPlatform` в тестах — подмена платформенного канала без наследования prod-кода.

### Сравнение в одной таблице

| | `extends` | `implements` |
|---|-----------|--------------|
| Количество (для классов) | Один суперкласс | Много `implements` |
| Реализация родителя | Наследуется | Не наследуется |
| `super` | Да | Нет (нет родительской реализации) |
| Типовые отношения | Подтип (`Dog` is `Animal`) | Подтип интерфейса (`MemoryStorage` is `Storage`) |
| Типичный кейс | Виджеты, базовые BLoC | Контракты, моки, адаптеры |

### Можно комбинировать

```dart
abstract class Entity {
  String get id;
}

class TimestampedEntity implements Entity {
  TimestampedEntity(this.id, this.createdAt);
  @override
  final String id;
  final DateTime createdAt;
}

class User extends TimestampedEntity {
  User(super.id, super.createdAt, this.name);
  final String name;
}
```

Здесь `User` **расширяет** класс с уже готовой имплементацией `Entity`, а не `implements Entity` напрямую.

### Пример из Flutter: зачем `implements` в тестах

```dart
// production
abstract class CartRepository {
  Future<int> itemCount();
}

// test — без сети и Firebase
class MockCartRepository implements CartRepository {
  int _count = 0;

  @override
  Future<int> itemCount() async => _count;

  void setCount(int n) => _count = n;
}
```

Если бы тест делал `extends` на concrete-класс с сетевыми вызовами, мок «унаследовал» бы лишнее поведение — нарушение подстановки Liskov и хрупкие тесты.

### Когда что выбирать

*   **`extends`** — общая реализация, шаблонный метод, базовый виджет/контроллер.
*   **`implements`** — граница модуля, несколько ролей у одного класса, подмена в DI.
*   **Оба с `abstract class`** — база с частичной реализацией: наследники `extends`, чужие модули — `implements` того же типа.

```dart
abstract class BaseBloc<Event, State> {
  void onEvent(Event event) { /* общий dispatch */ }
  void emit(State state) { /* ... */ }
}

class AuthBloc extends BaseBloc<AuthEvent, AuthState> {
  // наследуем каркас
}
```

## Что почитать

* [Dart: Extend a class](https://dart.dev/language/extend)
* [Dart: Classes — Implicit interfaces](https://dart.dev/language/classes#implicit-interfaces)
* [Dart language spec: extends](https://spec.dart.dev/language/extend)
* [Effective Dart: Design — Prefer defining constructors than static methods](https://dart.dev/effective-dart/design)
