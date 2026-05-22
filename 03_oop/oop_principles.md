# Расскажи принципы ООП. Как знание этих принципов помогает тебе в работе?

> ООП строится на четырёх классических столпах — **инкапсуляция**, **наследование**, **полиморфизм** и **абстракция** — плюс практические принципы проектирования (часто их сводят к **SOLID**). В Dart/Flutter это не «теория ради теории»: инкапсуляция прячет детали виджетов и сервисов, наследование и миксины переиспользуют поведение, полиморфизм позволяет подменять реализации (репозиторий, тема, навигация), а абстракция отделяет контракт от деталей. Знание этих идей помогает писать тестируемый код, не ломать API при расширении и не превращать один экран в «бог-виджет».

## Разбор

В промышленной разработке на Flutter чаще всего опираются на четыре столпа ООП и пять принципов SOLID. Они дополняют друг друга: столпы описывают *механизмы* языка, SOLID — *как ими пользоваться*, чтобы код оставался гибким.

### Четыре столпа ООП

*   **Инкапсуляция** — скрытие внутреннего состояния и выдача наружу только нужного API.
    *   *Аналогия:* автомобиль — вы крутите руль и жмёте педаль, не зная, как именно ECU управляет впрыском.
    *   *В Dart:* приватные члены через `_`, публичные геттеры, отдельные классы для бизнес-логики вместо всего в `State`.
    *   *Во Flutter:* `State` хранит приватные поля, наружу отдаёт только `build()`; `ChangeNotifier` не раскрывает, *как* он обновил подписчиков.

*   **Наследование** — построение иерархии «общее → частное», переиспользование реализации.
    *   *В Dart:* `extends` (один суперкласс), `with` для миксинов.
    *   *Во Flutter:* `StatelessWidget` / `StatefulWidget`, кастомные `RenderObject`, базовые классы в пакетах (`Bloc`, `Cubit`).

*   **Полиморфизм** — один и тот же вызов может работать с разными реализациями.
    *   *В Dart:* переменная типа `Animal` может ссылаться на `Dog` или `Cat`; переопределение `@override`.
    *   *Во Flutter:* экран принимает `abstract class AuthRepository` — в prod подставляется API, в тестах — mock.

*   **Абстракция** — выделение существенного контракта и отсечение деталей.
    *   *В Dart:* `abstract class`, `abstract interface class`, sealed-иерархии для моделей UI-состояния.
    *   *Во Flutter:* вы описываете *что* нужно от слоя данных, не *как* устроен HTTP-клиент.

### SOLID в контексте Flutter

| Принцип | Суть | Пример во Flutter |
|---------|------|-------------------|
| **S** — Single Responsibility | У класса одна причина для изменений | Отдельно: `UserRepository`, `LoginBloc`, `LoginPage` — не один виджет на 800 строк |
| **O** — Open/Closed | Расширяем без правки старого кода | Новый способ оплаты — новый класс `implements PaymentMethod`, а не `if/else` в процессоре |
| **L** — Liskov Substitution | Подтип можно подставить вместо базового | Любая реализация `ImageCache` должна вести себя предсказуемо для `Image` |
| **I** — Interface Segregation | Узкие контракты лучше «толстых» | `Readable` и `Writable` отдельно, а не один `Storage` с пустыми `@override` |
| **D** — Dependency Inversion | Зависимость от абстракций, не от concrete | `GetIt` / `Provider` отдают интерфейс репозитория, а не `FirebaseUserService` напрямую |

### Как это помогает в ежедневной работе

*   **Меньше связности.** Экран не знает про Dio/Retrofit — только про контракт репозитория → проще unit-тесты и смена бэкенда.
*   **Предсказуемые изменения.** Добавление нового состояния загрузки в sealed-модель заставляет компилятор напомнить обновить `switch` в UI — меньше «забытых веток».
*   **Читаемая архитектура.** Новый разработчик видит слои: domain (абстракции) → data (реализации) → presentation (виджеты).
*   **Рефакторинг без страха.** `implements` для моков, `extends` там, где нужна общая реализация — осознанный выбор, а не случайность.

### Пример: «бог-виджет» vs разделение ответственности

Плохой вариант — всё в одном `StatefulWidget`: HTTP, парсинг, валидация, навигация.

```dart
// Антипаттерн: один класс делает всё
class LoginScreen extends StatefulWidget { /* ... */ }

class _LoginScreenState extends State<LoginScreen> {
  Future<void> _login() async {
    final response = await http.post(Uri.parse('https://api.example/login'));
    // парсинг, setState, Navigator, SnackBar — всё здесь
  }
}
```

Лучше — инкапсуляция + абстракция + инверсия зависимостей:

```dart
abstract class AuthRepository {
  Future<User> signIn(String email, String password);
}

class LoginCubit {
  LoginCubit(this._auth);
  final AuthRepository _auth;

  Future<void> submit(String email, String password) async {
    // одна ответственность: состояние экрана входа
    emit(LoginLoading());
    try {
      final user = await _auth.signIn(email, password);
      emit(LoginSuccess(user));
    } catch (e) {
      emit(LoginFailure(e));
    }
  }
}

// UI только рисует по состоянию: UI = f(state)
class LoginPage extends StatelessWidget {
  const LoginPage({super.key, required this.cubit});
  final LoginCubit cubit;

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<LoginCubit, LoginState>(
      builder: (context, state) => switch (state) {
        LoginLoading() => const CircularProgressIndicator(),
        LoginSuccess(:final user) => Text('Hello, ${user.name}'),
        LoginFailure(:final error) => Text(error.toString()),
        _ => const LoginForm(),
      },
    );
  }
}
```

Здесь видны сразу несколько столпов: **инкапсуляция** (логика в Cubit), **абстракция** (`AuthRepository`), **полиморфизм** (подмена реализации в тестах), **SRP** и **DIP** из SOLID.

## Что почитать

* [Dart: Classes](https://dart.dev/language/classes)
* [Effective Dart: Design](https://dart.dev/effective-dart/design)
* [The SOLID Principles in Dart (DEV Community)](https://dev.to/actocodes/the-solid-principles-in-dart-building-robust-flutter-apps-4mho)
* [Flutter architectural samples (Bloc / layered architecture)](https://bloclibrary.dev/#/architecture)
