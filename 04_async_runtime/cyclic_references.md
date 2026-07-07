# Почему циклические ссылки не приводят к утечкам памяти в Dart?

> Потому что [сборщик мусора](garbage_collector.md) Dart использует **tracing GC** (обход от корней), а не подсчёт ссылок (reference counting). Tracing GC находит живые объекты, начиная от корней (стек, глобальные переменные), и обходит граф ссылок. Объекты, до которых невозможно добраться от корней, считаются мусором — даже если они ссылаются друг на друга. Циклическая ссылка A→B→A не спасает объекты от сборки, если ни A, ни B не достижимы от корня.

## Разбор

### Reference Counting vs Tracing GC

**Reference Counting** (Swift, Objective-C, CPython):
- Каждый объект хранит счётчик ссылок.
- Когда счётчик = 0, объект удаляется.
- **Проблема:** A→B→A — оба счётчика = 1, но объекты недостижимы → утечка.
- Решение: `weak` ссылки (Swift), ручной `weakref` (Python).

**Tracing GC** (Dart, Java, Go, C#):
- GC стартует от корней и обходит граф.
- Всё, что не посетил — мусор.
- Циклы **не проблема** по определению.

```
Корни (стек, глобалы)
    │
    ▼
  ┌───┐     ┌───┐
  │ X │────►│ Y │    ← живые (достижимы от корня)
  └───┘     └───┘
  
  ┌───┐     ┌───┐
  │ A │────►│ B │    ← мёртвые (цикл, но не достижимы)
  │   │◄────│   │
  └───┘     └───┘
```

### Пример на Dart

```dart
class Node {
  Node? next;
  final String name;
  Node(this.name);
}

void createCycle() {
  final a = Node('A');
  final b = Node('B');

  // Создаём цикл: A → B → A
  a.next = b;
  b.next = a;
  
  // Когда createCycle() завершается:
  // - локальные переменные a, b уходят со стека
  // - A и B больше не достижимы от корней
  // - GC при scavenge НЕ скопирует их в To-space → память освобождена
  // - Цикл A→B→A не мешает сборке
}
```

### Почему в Swift/Objective-C это проблема

```swift
// Swift (Reference Counting)
class ViewController {
    var closure: (() -> Void)?
    
    func setup() {
        // ⚠️ closure удерживает self, self удерживает closure → цикл → утечка
        closure = { 
            self.doSomething() 
        }
    }
    
    // Решение: [weak self]
    func setupFixed() {
        closure = { [weak self] in
            self?.doSomething()
        }
    }
}
```

В **Dart** аналогичный код **не течёт**:

```dart
class MyWidget extends StatefulWidget { /* ... */ }

class _MyWidgetState extends State<MyWidget> {
  VoidCallback? callback;

  @override
  void initState() {
    super.initState();
    // Замыкание захватывает this → цикл State↔Closure
    // Но это НЕ утечка: когда виджет удалён из дерева,
    // State и closure становятся недостижимы → GC их соберёт
    callback = () {
      setState(() {});
    };
  }
}
```

### Что тогда IS утечка в Dart?

Утечка в Dart — это когда объект **остаётся достижимым** от корня дольше, чем нужно:

```dart
// ⚠️ Утечка: подписка на глобальный стрим не отменена
class _LeakyState extends State<LeakyWidget> {
  @override
  void initState() {
    super.initState();
    // globalEventBus — глобальная переменная (корень!)
    // Подписка удерживает this → State не соберётся
    globalEventBus.listen((e) => setState(() {}));
  }
  // Нет dispose → нет cancel → утечка!
}

// ✅ Без утечки
class _SafeState extends State<SafeWidget> {
  late final StreamSubscription _sub;

  @override
  void initState() {
    super.initState();
    _sub = globalEventBus.listen((e) => setState(() {}));
  }

  @override
  void dispose() {
    _sub.cancel(); // Разрываем ссылку → State становится недостижим
    super.dispose();
  }
}
```

### Типичные «утечки» в Flutter (не из-за циклов!)

| Причина | Пример | Решение |
|---------|--------|---------|
| Неотменённая подписка | `stream.listen(...)` без `cancel()` | `dispose()` → `_sub.cancel()` |
| `Timer` не отменён | `Timer.periodic(...)` | `dispose()` → `_timer.cancel()` |
| `AnimationController` | Без `dispose()` | `dispose()` → `_controller.dispose()` |
| Глобальный кэш | `static Map<Key, Widget>` растёт | LRU-кэш с ограничением |
| `BuildContext` в async | `context` используется после `dispose` | Проверка `mounted` |

### Итого

В Dart **не нужны** `weak self`, `unowned`, `weakref` для борьбы с циклами. Но нужно **управлять временем жизни подписок** — именно это, а не циклические ссылки, вызывает утечки памяти во Flutter-приложениях.

## Что почитать

* [Dart: Memory leaks in Flutter](https://docs.flutter.dev/tools/devtools/memory#memory-leaks)
* [Dart VM: GC design](https://github.com/dart-lang/sdk/wiki/Garbage-Collection)
* [Understanding Garbage Collection in Dart (Medium)](https://medium.com/flutter-community/understanding-garbage-collection-in-dart-d7e7e58ad05f)
