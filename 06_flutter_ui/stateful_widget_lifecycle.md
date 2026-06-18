# Назови основные методы жизненного цикла StatefulWidget. Для чего нужен каждый из них?

> Жизненный цикл `StatefulWidget` — это последовательность вызовов вокруг объекта `State`: создание (`createState`), инициализация (`initState`), реакция на зависимости (`didChangeDependencies`), построение UI (`build`), обновление конфигурации (`didUpdateWidget`), временное удаление (`deactivate`), финальная очистка (`dispose`). Для обновления UI изнутри используют `setState`, а перед асинхронным `setState` проверяют `mounted`.

## Разбор

### Общая схема

```text
createState()
  -> initState()
  -> didChangeDependencies()
  -> build()
  -> (setState -> build)*
  -> didUpdateWidget() -> build()   // когда родитель передал новую конфигурацию
  -> deactivate()                   // виджет убрали из дерева
  -> dispose()                      // State больше не будет использоваться
```

### Основные методы и зачем они нужны

| Метод | Где объявлен | Когда вызывается | Для чего |
|-------|--------------|------------------|----------|
| `createState()` | `StatefulWidget` | При встраивании виджета в дерево | Создать объект `State`, который будет хранить изменяемые данные |
| `initState()` | `State` | Один раз после привязки `State` к `Element` | Первичная инициализация: контроллеры, подписки, локальные поля |
| `didChangeDependencies()` | `State` | После `initState` и при изменении `InheritedWidget`-зависимостей | Реакция на изменения данных «сверху» (тема, локаль, медиа-данные) |
| `build()` | `State` | После `setState`, изменения зависимостей, hot reload, `didUpdateWidget` | Вернуть актуальное дерево виджетов |
| `setState()` | `State` | Когда нужно обновить UI после изменения полей `State` | Пометить `Element` как «грязный» и запланировать новый `build` |
| `didUpdateWidget()` | `State` | Родитель пересоздал `StatefulWidget` того же типа и `key` | Сравнить старую/новую конфигурацию (`oldWidget` vs `widget`) |
| `deactivate()` | `State` | `State` временно убрали из дерева | Подготовка к возможному повторному встраиванию (редко переопределяют) |
| `dispose()` | `State` | `State` окончательно удален из дерева | Освободить ресурсы: `dispose()` контроллеров, отмена подписок |
| `mounted` (геттер) | `State` | Проверка перед асинхронным обновлением | Убедиться, что `State` еще в дереве, иначе `setState` упадет |

### Кратко по каждому методу

**`createState()`**  
Создает `State` для конкретного места в дереве. Обычно выглядит одинаково:

```dart
@override
State<Counter> createState() => _CounterState();
```

**`initState()`**  
Однократный старт: инициализация контроллеров, таймеров, локальных полей.  
Важно: первой строкой вызывать `super.initState()`.  
Не стоит здесь читать `InheritedWidget` через `of(context)` — для этого есть `didChangeDependencies`.

**`didChangeDependencies()`**  
Срабатывает после `initState` и когда меняются inherited-зависимости (например, тема/локаль).  
Здесь уже безопасно использовать `Theme.of(context)`, `MediaQuery.of(context)` и т.п.

**`build()`**  
Главный метод отрисовки: возвращает конфигурацию UI.  
Может вызываться часто, поэтому в `build` не делают тяжелую бизнес-логику, сетевые запросы и аналитику.

**`setState()`**  
Сигнал «состояние изменилось, пересобери UI».  
Внутри меняют поля `State`, после чего Flutter вызывает `build`.  
`setState` перерисовывает только тот `State`, где он был вызван.

**`didUpdateWidget()`**  
Нужен, когда родитель передал новый экземпляр того же виджета (тот же тип + `key`).  
Типичный кейс: сравнить `oldWidget.enabled` и `widget.enabled`, запустить/остановить анимацию.

**`deactivate()`**  
Вызывается при удалении из дерева. Если виджет быстро вернут в дерево (например, при `GlobalKey`), `State` может переиспользоваться.  
На практике переопределяют редко, но метод важен для понимания полного цикла.

**`dispose()`**  
Финальная очистка: `controller.dispose()`, `subscription.cancel()`, закрытие stream/timer.  
После `dispose` `mounted == false`, и `setState` вызывать нельзя.

**`mounted`**  
Проверка перед отложенным обновлением:

```dart
Future.delayed(const Duration(seconds: 2), () {
  if (!mounted) return;
  setState(() => _status = 'done');
});
```

### Мини-пример жизненного цикла

```dart
class LifecycleDemo extends StatefulWidget {
  const LifecycleDemo({required this.title, super.key});

  final String title;

  @override
  State<LifecycleDemo> createState() => _LifecycleDemoState();
}

class _LifecycleDemoState extends State<LifecycleDemo> {
  late final TextEditingController _controller;
  String _status = 'idle';

  @override
  void initState() {
    super.initState();
    _controller = TextEditingController();
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    // безопасно читать зависимости из context
    final brightness = Theme.of(context).brightness;
    _status = 'theme: $brightness';
  }

  @override
  void didUpdateWidget(covariant LifecycleDemo oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (oldWidget.title != widget.title) {
      _controller.text = widget.title;
    }
  }

  Future<void> _simulateAsyncUpdate() async {
    await Future<void>.delayed(const Duration(milliseconds: 400));
    if (!mounted) return;
    setState(() => _status = 'updated');
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        TextField(controller: _controller),
        Text(_status),
        ElevatedButton(
          onPressed: () {
            setState(() => _status = 'clicked');
            _simulateAsyncUpdate();
          },
          child: const Text('Run'),
        ),
      ],
    );
  }

  @override
  void deactivate() {
    // вызывается, когда State временно убран из дерева
    super.deactivate();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}
```

### Частые ошибки

1. Вызывать `setState` в `build` (ошибка `setState() called during build`).
2. Не освобождать ресурсы в `dispose` (утечки памяти).
3. Делать `setState` после `await` без проверки `mounted`.
4. Путать `didUpdateWidget` (новая конфигурация от родителя) и `setState` (локальное изменение `State`).

## Что почитать

* [Виджеты: StatefulWidget — жизненный цикл](https://education.yandex.ru/handbook/flutter/article/vidzheti-statefulwidget#zhiznennyj-cikl-statefulwidget)
* [State class (Flutter API)](https://api.flutter.dev/flutter/widgets/State-class.html)
* [StatefulWidget class (Flutter API)](https://api.flutter.dev/flutter/widgets/StatefulWidget-class.html)
* [Responding to widget lifecycle events](https://docs.flutter.dev/ui#responding-to-widget-lifecycle-events)
