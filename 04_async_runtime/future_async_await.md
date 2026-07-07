# Что такое Future? В чем разница между async/await и использованием метода .then()? Что из этого ты предпочитаешь и почему?

> `Future<T>` — объект, представляющий **отложенный результат** вычисления, которое завершится в будущем значением типа `T` или ошибкой. `async/await` превращает асинхронный код в линейную запись, похожую на синхронный — компилятор разбивает тело функции на «продолжения» (continuations) через state-machine. `.then()` — callback-стиль, где вы явно передаёте функцию-обработчик. На практике предпочтительнее `async/await`: код читается сверху вниз, ошибки ловятся обычным `try/catch`, а вложенность не растёт.

## Разбор

### Что внутри Future

При создании `Future` Dart помещает задачу в [Event Loop](event_loop.md). Когда задача завершается, Future переходит из состояния _uncompleted_ в одно из двух: **completed with value** или **completed with error**. Подписчики (через `.then`, `.catchError` или `await`) получают результат как [микрозадача](event_loop.md).

```dart
// Future может быть создан разными способами:
final f1 = Future.value(42);                       // уже завершён с значением
final f2 = Future.error('oops');                    // уже завершён с ошибкой
final f3 = Future(() => heavyComputation());        // выполнится в следующем event-loop цикле
final f4 = Future.delayed(Duration(seconds: 1));    // таймер + event queue
final f5 = http.get(Uri.parse('https://api.dev')); // I/O через native runtime
```

### async/await — линейный стиль

`async` помечает функцию как возвращающую `Future`. `await` приостанавливает выполнение функции до завершения Future, а затем возобновляет с результатом. Под капотом компилятор генерирует конечный автомат (state machine), где каждый `await` — точка переключения состояния.

```dart
Future<User> loadProfile(String id) async {
  try {
    final response = await apiClient.get('/users/$id');  // «пауза» 1
    final json = jsonDecode(response.body);
    final avatar = await storage.download(json['avatar']); // «пауза» 2
    return User.fromJson(json, avatar: avatar);
  } on SocketException {
    throw ServerFailure('Нет сети');
  } on FormatException {
    throw ParseFailure('Невалидный JSON');
  }
}
```

Преимущества:
* Читается как синхронный код — сверху вниз.
* Ошибки ловятся стандартным `try/catch/finally`.
* Легко отлаживать — стек-трейс показывает «точку await».

### .then() — callback-стиль

```dart
Future<User> loadProfile(String id) {
  return apiClient.get('/users/$id').then((response) {
    final json = jsonDecode(response.body);
    return storage.download(json['avatar']).then((avatar) {
      return User.fromJson(json, avatar: avatar);
    });
  }).catchError((e) {
    if (e is SocketException) throw ServerFailure('Нет сети');
    throw ParseFailure('Невалидный JSON');
  });
}
```

Уже при двух последовательных операциях виден **callback hell** — вложенность растёт, читаемость падает.

### Сравнительная таблица

| Критерий | `async/await` | `.then()` |
|----------|---------------|-----------|
| Читаемость | Линейная, как синхронный код | Вложенные callbacks |
| Обработка ошибок | `try/catch/finally` | `.catchError()` + `.whenComplete()` |
| Отладка | Stack trace указывает на `await` | Stack trace обрывается на callback |
| Возврат значения | `return value;` | `return future.then((_) => value);` |
| Условная логика | Обычный `if/else` | Ветвление через вложенные `.then` |
| Параллельный запуск | `await Future.wait([a, b])` | `Future.wait([a, b]).then(...)` |
| Циклы | `for (var x in items) await process(x);` | Рекурсивная цепочка `.then` |

### Когда .then() уместен

`.then()` выигрывает в редких ситуациях «fire-and-forget» или одношаговых трансформаций, когда создавать `async`-функцию — overkill:

```dart
// Одношаговая трансформация — async/await избыточен
Future<int> userCount() => fetchUsers().then((list) => list.length);

// Fire-and-forget: запустили и не ждём
void logAnalytics(Event event) {
  analytics.send(event).catchError((e) => logger.warning(e));
}
```

### Что предпочитаю и почему

В 95% случаев — **`async/await`**. Причины:

1. **Читаемость:** новый разработчик читает код сверху вниз, не разбирая цепочку колбэков.
2. **Ошибки:** `try/catch` ловит и синхронные, и асинхронные исключения в одном блоке.
3. **Рефакторинг:** добавить шаг между двумя `await` — одна строка, а не перестройка вложенности `.then`.
4. **Dart team recommendation:** официальный стиль [Effective Dart](https://dart.dev/effective-dart/usage#prefer-asyncawait-over-using-raw-futures) рекомендует `async/await`.

## Что почитать

* [Dart: Asynchronous programming — futures, async, await](https://dart.dev/codelabs/async-await)
* [Effective Dart: PREFER async/await over using raw futures](https://dart.dev/effective-dart/usage#prefer-asyncawait-over-using-raw-futures)
* [Dart: Future class API](https://api.dart.dev/stable/dart-async/Future-class.html)
