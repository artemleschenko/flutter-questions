# В чем разница между оператором == и идентичностью identical()? Зачем нужен пакет equatable и как он работает внутри?

> Функция `identical()` проверяет ссылочное равенство (указывают ли две переменные на один и тот же адрес в памяти), в то время как оператор `==` проверяет равенство значений (эквивалентность состояний объектов). Пакет `equatable` избавляет разработчика от рутинного и подверженного ошибкам ручного переопределения методов `==` и `hashCode`, вычисляя их автоматически под капотом на основе переданного списка полей (геттер props).

## Разбор

### Базовая механика `identical()` и `==`

В корневом классе Object в Dart оператор `==` по умолчанию реализован просто как вызов функции `identical(this, other)`. То есть, по умолчанию все объекты сравниваются по ссылке.
Чтобы объекты (например, модели данных `User(id: 1, name: 'John')`) сравнивались по их содержимому (Value Equality), вы обязаны переопределить оператор `==`.

### Золотое правило `hashCode`

Контракт Dart гласит: если вы переопределяете `==`, вы обязаны переопределить `hashCode`. Если`a == b` возвращает `true`, то `a.hashCode == b.hashCode` также обязано возвращать true. В противном случае эти объекты будут некорректно работать в коллекциях на основе хеш-таблиц (`Map`, `Set`), что приведет к скрытым багам (утечкам памяти, дубликатам).

### Как equatable работает под капотом?

Ручное переопределение `==` и `hashCode` для класса из 5-10 полей требует написания громоздкого "boilerplate" кода. Особенно сложно сравнивать вложенные коллекции (массивы не равны друг другу по значению по умолчанию, нужен `ListEquality`).Класс, наследующийся от `Equatable`, требует реализовать только один геттер: `List<Object?> get props`.

Внутренняя реализация `equatable`:

- Сравнение (`==`): Под капотом `equatable` берет список props обоих объектов и поэлементно сравнивает их. Если свойство является коллекцией (например, `List` или `Map`), `equatable` автоматически применяет глубокое сравнение (Deep Equality) с помощью утилит из пакета `collection` (например, `DeepCollectionEquality`).
- Хеширование (`hashCode`): `equatable` берет все элементы из `props` и пропускает их через алгоритм смешивания хешей (исторически — Jenkins Hash Function, в современных версиях часто делегируется к оптимизированному `Object.hashAll()`). Это гарантирует равномерное распределение хешей и минимизацию коллизий со сложностью O(N), где N — количество свойств.

```dart
// Без equatable (много кода, высокий риск забыть обновить методы при добавлении поля)
class User {
  final int id;
  final List<String> roles;
  
  User(this.id, this.roles);

  @override
  bool operator ==(Object other) {
    if (identical(this, other)) return true;
    final listEquals = const DeepCollectionEquality().equals;
    return other is User && other.id == id && listEquals(other.roles, roles);
  }

  @override
  int get hashCode => id.hashCode ^ const DeepCollectionEquality().hash(roles);
}

// С equatable
import 'package:equatable/equatable.dart';

class EquatableUser extends Equatable {
  final int id;
  final List<String> roles;

  const EquatableUser(this.id, this.roles);

  // Под капотом автоматически генерируются корректные == и hashCode,
  // включая глубокое сравнение списка roles.
  @override
  List<Object?> get props => [id, roles]; 
}
```

### Практическое применение

В production-проектах equatable — это индустриальный стандарт при работе с архитектурными паттернами и стейт-менеджментом, особенно BLoC (Business Logic Component).

Метод `emit(newState)` в Bloc сравнивает текущее состояние с новым. Если `currentState == newState`, BLoC игнорирует событие и не перерисовывает UI. Если ваши классы состояний (States) не наследуют `Equatable`, каждый новый экземпляр класса будет иметь уникальную ссылку в памяти (ссылочное неравенство), BLoC посчитает состояние измененным, и фреймворк выполнит ненужный, ресурсоемкий `rebuild` виджетов дерева, вызывая просадки FPS.

## Что почитать

* [identical function](https://api.dart.dev/dart-core/identical.html)
* [operator == method](https://api.dart.dev/dart-core/Object/operator_equals.html)
* [GitHub equatable](https://github.com/felangel/equatable/blob/master/lib/src/equatable.dart)
