# Объясни своими словами, чем отличается абстрактный класс от обычного интерфейса в реалиях Dart (учитывая, что ключевого слова interface до Dart 3 не было)?

> В Dart **каждый класс неявно объявляет интерфейс** — набор публичных instance-членов. До Dart 3 отдельного `interface` не было: «интерфейс» получали через `implements SomeClass`, а **абстрактный класс** (`abstract class`) — это класс, который нельзя инстанцировать и в котором *можно* оставить методы без тела. Разница на практике: `extends` тянет реализацию и иерархию «is-a», `implements` — только контракт «умеет то же API», со своей реализацией с нуля. С Dart 3 появился модификатор `interface` и связка `abstract interface class` — это уже *явный* чистый контракт, ближе к Java/C# interface.

## Разбор

Путаница возникает из-за того, что в Java «interface» — отдельная сущность, а в Dart долгое время **интерфейсом была любая публичная поверхность класса**.

### Неявный интерфейс (до и после Dart 3)

Официальная формулировка: каждый класс неявно определяет интерфейс из своих instance-членов и интерфейсов, которые он уже реализует.

```dart
class Person {
  final String _name;
  Person(this._name);
  String greet(String who) => 'Hello, $who. I am $_name.';
}

// «Интерфейс Person» без отдельного файла — просто implements Person
class Bot implements Person {
  @override
  String greet(String who) => 'Hi $who, I am a bot.';
}
```

`Impostor` не наследует `_name` и конструктор `Person` — он **обязан** сам реализовать публичный API (здесь — `greet`). Приватные члены (`_name`) в интерфейс не входят.

### Абстрактный класс

`abstract class` нельзя создать через конструктор (если нет factory), зато можно:

* наследовать (`extends`) — с доработкой/переопределением;
* реализовать (`implements`) — как неявный интерфейс.

```dart
abstract class Shape {
  double get area; // абстрактный getter — тела нет

  void describe() => print('Shape with area $area'); // общая реализация
}

class Circle extends Shape {
  Circle(this.radius);
  final double radius;

  @override
  double get area => 3.14 * radius * radius;
}
```

**Своими словами:** абстрактный класс — «черновик семьи объектов»: часть поведения уже написана, часть — заставляет наследников дописать. Неабстрактный класс без `abstract` на методах не может оставлять методы без реализации.

### «Обычный интерфейс» до Dart 3

Интерфейсом называли не тип с ключевым словом, а **способ использования**:

| Подход | Что получаете | Когда уместно |
|--------|---------------|---------------|
| `extends AbstractRepo` | Реализацию родителя + обязанность дописать абстрактное | Общая логика в базе (логирование, кэш) |
| `implements AbstractRepo` | Только контракт, **без** унаследованного кода | Несколько «ролей», моки, адаптеры к чужому API |

```dart
abstract class UserRepository {
  Future<User> getUser(String id);
}

// Наследование: можно переиспользовать protected/общие методы базы
class CachedUserRepository extends UserRepository {
  @override
  Future<User> getUser(String id) async { /* ... */ }
}

// Имплементация: «форма» API, тело пишем с нуля
class InMemoryUserRepository implements UserRepository {
  final _store = <String, User>{};

  @override
  Future<User> getUser(String id) async => _store[id]!;
}
```

### Dart 3: модификатор `interface` — не то же самое, что «старый интерфейс»

`interface class` **запрещает `extends` вне библиотеки**, но разрешает `implements`. Чистый контракт без наследования реализации:

```dart
// lib/domain/payment.dart
abstract interface class PaymentGateway {
  Future<void> charge(Money amount);
}

// lib/data/stripe_gateway.dart
class StripeGateway implements PaymentGateway {
  @override
  Future<void> charge(Money amount) async { /* Stripe SDK */ }
}
```

Связка **`abstract interface class`** — канонический «чистый интерфейс» в Dart 3: нельзя инстанцировать, нельзя наследовать снаружи, можно только `implements`.

### Аналогия для Flutter-разработчика

* **Абстрактный класс** — как `StatefulWidget`: общий каркас (`createState`), детали в наследнике.
* **Неявный/явный интерфейс** — как контракт `Listenable`: `AnimatedBuilder` не знает, *как* устроен ваш `ChangeNotifier`, ему важен только API `addListener` / `removeListener`.

В `flutter_test` вы постоянно `implements` или подменяете каналы (`TestDefaultBinaryMessengerBinding`) — это имплементация контракта, а не расширение реализации фреймворка.

### Частая ошибка на собеседовании

> «В Dart нет интерфейсов» — **неверно**. Не было отдельного *ключевого слова* до Dart 3; интерфейс был у каждого класса. С Dart 3 слово `interface` вернулось, но с **другой семантикой**, чем в Java: это модификатор доступа к наследованию, а не отдельный тип файла.

## Что почитать

* [Dart: Classes — Implicit interfaces](https://dart.dev/language/classes#implicit-interfaces)
* [Dart: Class modifiers — abstract, interface](https://dart.dev/language/class-modifiers)
* [Dart: Abstract methods](https://dart.dev/language/methods#abstract-methods)
* [Stack Overflow: Dart difference between implicit interface and abstract class](https://stackoverflow.com/questions/70565993/dart-difference-between-implicit-interface-and-abstract-class)
