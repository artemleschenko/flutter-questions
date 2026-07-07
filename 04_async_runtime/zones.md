# Что такое Zone в Dart и как этот механизм используется для перехвата необработанных ошибок?

> `Zone` — изолированный контекст исполнения, который перехватывает и переопределяет поведение асинхронного кода: обработку ошибок, создание [Timer](event_loop.md)'ов, `print()`, `scheduleMicrotask()` и запуск [Future](future_async_await.md)/[Stream](stream_internals.md). Основное применение — «ловить» необработанные ошибки из асинхронных операций, которые не были обёрнуты в `try/catch`. Flutter использует `runZonedGuarded` для отправки crash-отчётов в Crashlytics/Sentry без падения приложения.

## Разбор

### Проблема, которую решает Zone

```dart
// ⚠️ try/catch НЕ ловит ошибку из Future, запущенного «в пустоту»
void main() {
  try {
    Future(() => throw Exception('boom')); // fire-and-forget
  } catch (e) {
    print('Caught: $e'); // НИКОГДА не выполнится!
  }
  // Exception улетает в «пустоту» → приложение может упасть
}
```

`try/catch` работает синхронно. [Future](future_async_await.md), запущенный без `await`, завершается позже — ошибка не перехвачена. Zone решает это.

### runZonedGuarded — основной инструмент

```dart
void main() {
  runZonedGuarded(
    () {
      // Весь код внутри этой Zone
      runApp(const MyApp());
    },
    (Object error, StackTrace stack) {
      // Сюда попадают ВСЕ необработанные ошибки из зоны
      // — незапертые Future
      // — ошибки в Stream без onError
      // — throw без catch в async-коде
      crashlytics.recordError(error, stack);
      logger.error('Unhandled: $error', stack);
    },
  );
}
```

### Архитектура Zone

```
┌────────────────────────────────────────────────┐
│                Root Zone                        │
│  (Zone.root — глобальная, без обработки ошибок) │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │           Custom Zone (runZonedGuarded)   │   │
│  │                                           │   │
│  │  onError: (e, st) => logToCrashlytics()  │   │
│  │  print: (zone, delegate, msg) => ...     │   │
│  │  fork: (zone, delegate, spec) => ...     │   │
│  │                                           │   │
│  │  ┌───────────────────────────────────┐   │   │
│  │  │     Child Zone (тест / виджет)    │   │   │
│  │  │  Наследует обработчики от родителя │   │   │
│  │  └───────────────────────────────────┘   │   │
│  └──────────────────────────────────────────┘   │
└────────────────────────────────────────────────┘
```

Zone'ы образуют **дерево**. Каждая дочерняя Zone наследует обработчики родительской, если не переопределяет их.

### Полная инициализация Flutter-приложения

```dart
void main() {
  // 1. Zone перехватывает все асинхронные ошибки
  runZonedGuarded(
    () async {
      WidgetsFlutterBinding.ensureInitialized();

      // 2. FlutterError.onError — ошибки фреймворка (build, layout, paint)
      FlutterError.onError = (details) {
        FlutterError.presentError(details); // красный экран в debug
        crashlytics.recordFlutterFatalError(details);
      };

      // 3. PlatformDispatcher — ошибки, не пойманные ни Zone, ни Flutter
      PlatformDispatcher.instance.onError = (error, stack) {
        crashlytics.recordError(error, stack, fatal: true);
        return true; // true = ошибка обработана, не падаем
      };

      await Firebase.initializeApp();
      runApp(const CityMonitorApp());
    },
    (error, stack) {
      // 4. Необработанные async-ошибки (fire-and-forget Future и т.д.)
      crashlytics.recordError(error, stack);
    },
  );
}
```

### Какие ошибки куда попадают

| Источник ошибки | Обработчик |
|-----------------|------------|
| `throw` в `build()` | `FlutterError.onError` |
| Исключение в `layout()` / `paint()` | `FlutterError.onError` |
| `throw` в `async` без `try/catch` + без `await` | `Zone.onError` (runZonedGuarded) |
| Ошибка в [Stream](stream_internals.md) без `onError` | `Zone.onError` |
| Ошибка в Platform Channel | `PlatformDispatcher.onError` |
| `Isolate` uncaught error | `Isolate.addErrorListener` |

### Zone как контекст — ZoneValues

Zone может хранить пары key-value, доступные из любого кода внутри:

```dart
final requestIdKey = Object(); // уникальный ключ

void handleRequest(String id) {
  runZoned(
    () {
      processStep1(); // Внутри processStep1 можно достать requestId
      processStep2();
    },
    zoneValues: {requestIdKey: id},
  );
}

void processStep1() {
  final id = Zone.current[requestIdKey]; // Доступ к значению без передачи аргументов
  print('Processing request $id in step 1');
}
```

> Во Flutter это используется для `debugPrint`, `Timeline`, и тестовых фреймворков.

### Zone в тестах

`flutter_test` создаёт Zone для каждого теста, чтобы:
- Перехватывать `print()` → `prints()` matcher.
- Перехватывать `Timer` → fake async.
- Ловить необработанные ошибки → `expect(fn, throwsA(...))`.

```dart
// package:fake_async использует Zone для управления временем
fakeAsync((async) {
  var called = false;
  Timer(const Duration(seconds: 10), () => called = true);
  
  async.elapse(const Duration(seconds: 9));
  expect(called, isFalse);
  
  async.elapse(const Duration(seconds: 1));
  expect(called, isTrue); // Timer «сработал» без реального ожидания
});
```

### Что Zone переопределяет (ZoneSpecification)

```dart
runZoned(
  () => myApp(),
  zoneSpecification: ZoneSpecification(
    // Перехват print
    print: (self, parent, zone, message) {
      parent.print(zone, '[APP] $message');
    },
    // Перехват создания Timer
    createTimer: (self, parent, zone, duration, callback) {
      print('Timer created for $duration');
      return parent.createTimer(zone, duration, callback);
    },
    // Перехват scheduleMicrotask
    scheduleMicrotask: (self, parent, zone, callback) {
      parent.scheduleMicrotask(zone, () {
        try {
          callback();
        } catch (e, st) {
          logger.error('Microtask failed', e, st);
        }
      });
    },
  ),
);
```

### Практические правила

1. **Всегда** используйте `runZonedGuarded` в `main()` для production-приложений.
2. **Не используйте** `Zone.current[key]` как замену DI — это implicit dependency, тяжело тестировать.
3. **Комбинируйте** три уровня: `runZonedGuarded` + `FlutterError.onError` + `PlatformDispatcher.onError`.
4. **В тестах** зоны работают автоматически — `flutter_test` оборачивает каждый тест.

## Что почитать

* [Dart: Zones](https://dart.dev/articles/zones)
* [Dart: Zone class API](https://api.dart.dev/stable/dart-async/Zone-class.html)
* [Flutter: Handling errors](https://docs.flutter.dev/testing/errors)
* [Flutter: Firebase Crashlytics setup](https://firebase.google.com/docs/crashlytics/get-started?platform=flutter)
