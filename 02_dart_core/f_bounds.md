# Что понимают под F-bounds в Dart? В каких случаях при проектировании базовых классов/архитектуры тебе понадобится использовать рекурсивные дженерики?

> F-bounds (F-ограниченный [полиморфизм](../03_oop/oop_principles.md)) или рекурсивные [дженерики](generics.md) — это паттерн проектирования системы типов, при котором параметр обобщения (generic) ограничен классом, содержащим этот же самый параметр (например, `abstract class Builder<T extends Builder<T>>`). Это позволяет базовому классу налагать строгий контракт на свои методы так, чтобы они принимали или возвращали строго конкретный производный класс (subclass), а не обобщенный базовый тип, решая проблему потери информации о типе при наследовании.

## Разбор

F-bounds (сокращение от F-bounded polymorphism, также известные как рекурсивные дженерики или рекурсивные ограничения типов) — это мощная концепция объектно-ориентированного программирования и системы типов Dart.

Говоря простым языком, F-bounds — это ситуация, когда параметр типа (generic) ограничен классом, который сам использует этот же параметр типа.

В Dart это выглядит так:

```dart
class BaseClass<T extends BaseClass<T>> { ... }
```

Эта конструкция чаще всего используется для решения проблемы потери типа при [наследовании](../03_oop/oop_principles.md), особенно когда вы хотите возвращать `this` (например, в паттерне Builder или при цепочках вызовов методов) или когда нужно строго типизировать операции сравнения и клонирования.

### Проблема: Потеря типа дочернего класса (Без F-bounds)

Представьте, что вы пишете паттерн Builder для создания объектов в Flutter/Dart. У вас есть базовый класс и наследник.

```dart
// БАЗОВЫЙ КЛАСС
class VehicleBuilder {
  String? engine;

  VehicleBuilder setEngine(String e) {
    engine = e;
    return this; // Возвращает тип VehicleBuilder
  }
}

// ДОЧЕРНИЙ КЛАСС
class CarBuilder extends VehicleBuilder {
  int? doors;

  CarBuilder setDoors(int d) {
    doors = d;
    return this; // Возвращает тип CarBuilder
  }
}

void main() {
  final builder = CarBuilder()
      .setDoors(4)     // Возвращает CarBuilder. Все отлично.
      .setEngine('V8') // Возвращает VehicleBuilder!
      .setDoors(2);    // ОШИБКА КОМПИЛЯЦИИ! У VehicleBuilder нет метода setDoors.
}
```

Что пошло не так? Метод `setEngine` определен в базовом классе и жестко возвращает `VehicleBuilder`. Как только мы вызываем его в цепочке (method chaining), компилятор «забывает», что на самом деле мы работаем с `CarBuilder`.

### Решение: Использование F-bounds

Чтобы заставить методы базового класса возвращать правильный тип дочернего класса, мы используем F-bounded polymorphism. Мы передаем тип наследника в качестве дженерика в базовый класс.

```dart
// 1. Указываем <T extends VehicleBuilder<T>>
class VehicleBuilder<T extends VehicleBuilder<T>> {
  String? engine;

  // 2. Возвращаем T вместо базового класса
  T setEngine(String e) {
    engine = e;
    return this as T; // 3. Делаем каст `this` к типу `T`
  }
}

// 4. При наследовании передаем СВОЙ ЖЕ ТИП в дженерик базового класса
class CarBuilder extends VehicleBuilder<CarBuilder> {
  int? doors;

  CarBuilder setDoors(int d) {
    doors = d;
    return this; 
  }
}

void main() {
  final builder = CarBuilder()
      .setEngine('V8') // Теперь setEngine возвращает `T` (то есть CarBuilder)
      .setDoors(4)     // Работает!
      .setEngine('V6'); // Снова работает!
      
  print('Двигатель: ${builder.engine}, Двери: ${builder.doors}');
}
```

Примечание про `this as T`: В Dart (в отличие от некоторых других языков) необходимо явно приводить `this` к `T`. Компилятор требует этого, так как теоретически кто-то мог бы написать `class BadBuilder extends VehicleBuilder<CarBuilder>` (передать чужой класс), и тогда `this` не было бы равно `T`. Приведение типов говорит компилятору: "Я гарантирую, что дочерний класс передаст свой собственный тип".

## Что почитать

* [Generics](https://dart.dev/language/generics)
* [Type system](https://dart.dev/language/type-system)
* [Bounded quantification](https://en.wikipedia.org/wiki/Bounded_quantification#F-bounded_quantification)
