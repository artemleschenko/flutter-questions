# Как в Dart перехватывать и обрабатывать ошибки при работе с Future (например, при сетевом запросе)?

> В `async`-функциях ошибки ловятся обычным `try/catch/finally`, где `catch` перехватывает и синхронные, и асинхронные исключения. В callback-стиле используется `.catchError()` и `.whenComplete()`. В Clean Architecture ошибки из Data-слоя (`SocketException`, `TimeoutException`, `FormatException`) нужно оборачивать в доменные `Failure`-объекты, чтобы UI не зависел от деталей сетевого клиента.

## Разбор

### Базовый перехват с async/await

```dart
Future<WeatherEntity> fetchWeather(String city) async {
  try {
    final response = await dio.get('/weather', queryParameters: {'q': city});
    return WeatherDto.fromJson(response.data).toEntity();
  } on DioException catch (e) {
    // Сетевые / HTTP ошибки Dio
    switch (e.type) {
      case DioExceptionType.connectionTimeout:
      case DioExceptionType.receiveTimeout:
        throw TimeoutFailure('Сервер не ответил вовремя');
      case DioExceptionType.badResponse:
        throw ServerFailure('HTTP ${e.response?.statusCode}');
      default:
        throw NetworkFailure('Нет подключения к сети');
    }
  } on FormatException {
    throw ParseFailure('Невалидный JSON от сервера');
  } finally {
    // Гарантированно выполнится — освобождение ресурсов
    logger.info('fetchWeather($city) завершён');
  }
}
```

Ключевой момент: `on Type catch (e)` позволяет фильтровать по типу исключения, а голый `catch (e, stackTrace)` — «поймать всё» для логирования.

### Callback-стиль (.catchError)

```dart
Future<String> fetchTitle() {
  return http
      .get(Uri.parse('https://api.dev/title'))
      .then((response) => jsonDecode(response.body)['title'] as String)
      .catchError(
        (Object e) => 'Fallback title',
        test: (e) => e is SocketException, // фильтр по типу
      )
      .whenComplete(() => logger.info('fetchTitle done'));
}
```

> **Ловушка:** `.catchError` не ловит синхронные ошибки внутри `.then()`, если [Future](future_async_await.md) уже завершён. `async/await` + `try/catch` свободен от этой проблемы.

### Паттерн Result (Either) — без исключений

Вместо `throw` можно возвращать sealed-класс, чтобы компилятор заставил обработать оба случая:

```dart
sealed class Result<T> {
  const Result();
}

final class Ok<T> extends Result<T> {
  const Ok(this.value);
  final T value;
}

final class Err<T> extends Result<T> {
  const Err(this.failure);
  final Failure failure;
}

// Использование в репозитории
Future<Result<User>> getUser(String id) async {
  try {
    final dto = await remoteSource.fetchUser(id);
    return Ok(dto.toEntity());
  } on DioException catch (e) {
    return Err(ServerFailure(e.message ?? 'Unknown'));
  }
}
```

UI обязан обработать оба варианта через `switch`:

```dart
final result = await getUser('123');
switch (result) {
  case Ok(:final value):
    showProfile(value);
  case Err(:final failure):
    showError(failure.localizedMessage(context));
}
```

### Доменные Failure-классы

В Clean Architecture ошибки не должны «просачиваться» в UI. Data-слой ловит низкоуровневые исключения и бросает/возвращает абстрактные Failure:

```dart
// domain/failures.dart
sealed class Failure {
  const Failure(this.message);
  final String message;
}

class ServerFailure extends Failure {
  const ServerFailure(super.message);
}

class NetworkFailure extends Failure {
  const NetworkFailure(super.message);
}

class CacheFailure extends Failure {
  const CacheFailure(super.message);
}
```

| Исходное исключение | Failure | Когда возникает |
|---------------------|---------|-----------------|
| `SocketException` | `NetworkFailure` | Нет интернета |
| `TimeoutException` | `ServerFailure` | Сервер не ответил |
| `FormatException` | `ParseFailure` | Битый JSON |
| `PostgrestException` | `ServerFailure` | Ошибка Supabase |
| `AuthException` | `AuthFailure` | Невалидный токен |

### Future.wait — параллельные запросы с обработкой ошибок

```dart
Future<Dashboard> loadDashboard() async {
  try {
    final (user, stats, notifications) = await (
      userRepo.current(),
      statsRepo.today(),
      notificationRepo.unread(),
    ).wait; // Dart 3 record destructuring
    return Dashboard(user: user, stats: stats, notifications: notifications);
  } on ParallelWaitError catch (e) {
    // Одна или несколько Future упали
    throw ServerFailure('Не удалось загрузить дашборд: ${e.errors}');
  }
}
```

### Необработанные ошибки

Если Future завершается ошибкой и никто не подписан — ошибка попадает в текущую [Zone](zones.md). В Flutter `FlutterError.onError` и `PlatformDispatcher.instance.onError` перехватывают такие «потерянные» ошибки для Crashlytics/логирования.

## Что почитать

* [Dart: Error handling](https://dart.dev/language/error-handling)
* [Effective Dart: DON'T discard errors from calls to catchError](https://dart.dev/effective-dart/usage#dont-discard-errors-from-calls-to-catcherror)
* [Dart: Future.wait API](https://api.dart.dev/stable/dart-async/Future/wait.html)
