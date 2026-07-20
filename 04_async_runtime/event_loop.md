# Как работает Event Loop? Как приоритетность очереди микрозадач (Microtasks) относительно обычных событий влияет на отзывчивость интерфейса?

> Dart — однопоточный язык с кооперативной [асинхронностью](future_async_await.md). Event Loop — бесконечный цикл, который последовательно обрабатывает задачи из двух очередей: **Microtask Queue** (высокий приоритет) и **Event Queue** (обычный приоритет). После каждого события цикл **полностью опустошает** микрозадачи, и только затем берёт следующее событие. Если микрозадач слишком много, Event Queue голодает — UI-фреймы не обрабатываются, и интерфейс замирает.

## Разбор

### Архитектура Event Loop

```
┌─────────────────────────────────────────────┐
│                 main()                       │
│   (синхронный код выполняется первым)        │
└──────────────────┬──────────────────────────┘
                   ▼
         ┌─────────────────┐
         │   Event Loop     │◄──────────────────────┐
         └────────┬────────┘                        │
                  ▼                                 │
     ┌────────────────────────┐                     │
     │ Microtask Queue пуста? │──нет──► Выполнить   │
     │                        │        микрозадачу ──┘
     └──────────┬─────────────┘
              да│
                ▼
     ┌────────────────────────┐
     │ Event Queue пуста?     │──нет──► Выполнить
     │                        │        событие ──► (назад к Microtask)
     └──────────┬─────────────┘
              да│
                ▼
          Ожидание (idle)
```

**Ключевое правило:** после каждого Event Loop берёт **все** накопившиеся микрозадачи до единой, и только потом — одно событие из Event Queue.

### Что попадает в какую очередь

| Microtask Queue | Event Queue |
|-----------------|-------------|
| `scheduleMicrotask()` | I/O callbacks (HTTP, файлы) |
| Завершение `Future` (`.then`, `await`) | `Timer`, `Future.delayed` |
| `Future.microtask()` | UI-события (тапы, жесты) |
| `Completer.complete()` | `Stream` события |
| | `SchedulerBinding.addPostFrameCallback` |

### Порядок выполнения — пример

```dart
void main() {
  print('1: main sync');

  Future(() => print('5: event queue (Future)'));

  Future.microtask(() => print('3: microtask 1'));

  scheduleMicrotask(() => print('4: microtask 2'));

  Future.delayed(Duration.zero, () => print('6: delayed (event queue)'));

  print('2: main sync end');
}

// Вывод:
// 1: main sync
// 2: main sync end
// 3: microtask 1
// 4: microtask 2
// 5: event queue (Future)
// 6: delayed (event queue)
```

**Логика:** синхронный код (`main`) → все микрозадачи → события из Event Queue в порядке добавления.

### Как это влияет на UI

Flutter рисует кадры примерно каждые **16.6 мс** (60 FPS). Отрисовка кадра — это событие в Event Queue. Если перед ним скопились микрозадачи, кадр «ждёт»:

```dart
// ⚠️ ПЛОХО: заблокирует UI
void processAllItems(List<Item> items) {
  for (final item in items) {
    // Каждый Future.microtask добавляет микрозадачу
    Future.microtask(() => heavyProcess(item));
  }
  // Все 10 000 микрозадач выполнятся ДО следующего кадра UI
}

// ✅ ХОРОШО: даём Event Loop «дышать»
void processAllItems(List<Item> items) {
  for (final item in items) {
    // Future() кладёт в Event Queue — UI-кадры чередуются
    Future(() => heavyProcess(item));
  }
}
```

### Реальный сценарий: `await` в цикле

```dart
// ⚠️ Подвох: каждый await порождает микрозадачу при resolve
Future<void> uploadPhotos(List<File> photos) async {
  for (final photo in photos) {
    await compress(photo);    // микрозадача на resolve
    await upload(photo);      // ещё микрозадача
    await updateProgress();   // и ещё
  }
  // Но тут ок: между await'ами Event Loop успевает вставить кадр,
  // потому что I/O (compress, upload) — это Event Queue
}
```

Настоящая проблема возникает, когда `await` разрешается **мгновенно** (например, `Future.value`) — тогда цепочка микрозадач не даёт вставить кадр.

### Flutter SchedulerBinding

Flutter использует приоритеты внутри Event Loop через `SchedulerBinding`:

```dart
// Выполнится ПОСЛЕ текущего build/layout/paint
WidgetsBinding.instance.addPostFrameCallback((_) {
  // Безопасно обращаться к RenderObject, размеры уже посчитаны
  scrollController.jumpTo(scrollController.position.maxScrollExtent);
});
```

### Практические правила

1. **Не злоупотребляйте `scheduleMicrotask`** — он вытесняет UI-кадры.
2. **Тяжёлые вычисления** отправляйте в `Isolate` через `compute()` или `Isolate.run()` — они работают в отдельном потоке и не блокируют Event Loop.
3. **Для пакетной обработки** используйте `Future()` (Event Queue), а не `Future.microtask()`.
4. **`Timer.periodic`** для анимаций / polling — даёт UI «дышать» между тиками.

## Что почитать

* [Dart: The Event Loop and Dart](https://dart.dev/language/concurrency#the-event-loop)
* [The Event Loop in Dart (Medium)](https://medium.com/dartlang/dart-asynchronous-programming-isolates-and-event-loops-bffc3e296a6a)
* [Flutter: SchedulerBinding](https://api.flutter.dev/flutter/scheduler/SchedulerBinding-mixin.html)
