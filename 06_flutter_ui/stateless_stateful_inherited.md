# В чем фундаментальная разница между StatelessWidget, StatefulWidget и InheritedWidget?

> Коротко: `StatelessWidget` — это **чистая декларация UI из входных данных**; `StatefulWidget` — это **декларация UI + связанный объект состояния**, который может меняться во времени; `InheritedWidget` — это **механизм передачи общих данных вниз по дереву** и подписки зависимых виджетов на изменения этих данных.

## Разбор

### В чем их разные роли

| Виджет | Главная роль | Где хранится изменяемое состояние | Когда использовать |
|--------|--------------|-----------------------------------|--------------------|
| `StatelessWidget` | Отрисовать UI из текущих параметров | Нигде внутри самого виджета | Экран/часть UI без локального mutable-state |
| `StatefulWidget` | Отрисовать UI, который меняется со временем | В отдельном объекте `State` | Когда нужны локальные изменения: ввод, анимации, таймер, toggle |
| `InheritedWidget` | Прокинуть данные глубоко по дереву без prop-drilling | В родителе, который его пересоздает | Когда данные нужны многим потомкам: тема, локаль, конфиг |

Фундаментальная идея: это не «конкурирующие» классы, а **разные уровни ответственности** в дереве UI.

### 1) StatelessWidget: UI как функция входных данных

`StatelessWidget` не хранит изменяемое состояние внутри себя. Он получает параметры и строит виджеты.

```dart
class UserBadge extends StatelessWidget {
  const UserBadge({required this.name, super.key});

  final String name;

  @override
  Widget build(BuildContext context) {
    return Text(name);
  }
}
```

Если `name` изменится, родитель вернет новый `UserBadge(name: ...)`, и UI обновится.

### 2) StatefulWidget: отдельная память между rebuild

`StatefulWidget` разделен на две части:
- сам виджет (иммутабельная конфигурация),
- `State` (изменяемые поля и логика обновления).

```dart
class Counter extends StatefulWidget {
  const Counter({super.key});

  @override
  State<Counter> createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  int _count = 0;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text('$_count'),
        ElevatedButton(
          onPressed: () => setState(() => _count++),
          child: const Text('Increment'),
        ),
      ],
    );
  }
}
```

`setState` не «меняет виджет магически», а помечает этот участок дерева на перестроение: `build()` вызывается заново и возвращает новую конфигурацию UI.

### 3) InheritedWidget: данные вниз по дереву

`InheritedWidget` нужен, когда значение требуется далеко внизу дерева, и не хочется прокидывать его через каждый конструктор.

```dart
class AppConfig extends InheritedWidget {
  const AppConfig({
    required this.apiHost,
    required super.child,
    super.key,
  });

  final String apiHost;

  static AppConfig of(BuildContext context) {
    final result = context.dependOnInheritedWidgetOfExactType<AppConfig>();
    assert(result != null, 'No AppConfig found in context');
    return result!;
  }

  @override
  bool updateShouldNotify(AppConfig oldWidget) => apiHost != oldWidget.apiHost;
}
```

Потомок может получить значение так: `final host = AppConfig.of(context).apiHost;`.
Если `apiHost` поменяется и родитель пересоздаст `AppConfig`, зависимые потомки обновятся.

### Ключевое отличие

- `StatelessWidget` отвечает на вопрос: **как отрисовать UI из текущих аргументов**.
- `StatefulWidget` отвечает на вопрос: **где и как хранить локальное состояние этого участка UI**.
- `InheritedWidget` отвечает на вопрос: **как передать общее состояние/данные многим потомкам без ручного прокидывания параметров**.

Их обычно используют вместе: верхний уровень дает данные через `InheritedWidget`, отдельные куски интерфейса остаются `StatelessWidget`, а локально интерактивные блоки делают `StatefulWidget`.

### Частые ошибки

1. Делать `StatefulWidget` «на всякий случай», когда достаточно `StatelessWidget`.
2. Складывать в `StatefulWidget` глобальные данные, нужные всему дереву.
3. Прокидывать параметры через много уровней вместо общего провайдера данных.
4. Забывать, что `BuildContext` должен быть из правильного места дерева для доступа к `InheritedWidget`.

## Что почитать

* [Building user interfaces with Flutter](https://docs.flutter.dev/ui)
* [Виджеты: StatefulWidget — Яндекс Образование](https://education.yandex.ru/handbook/flutter/article/vidzheti-statefulwidget)
* [Inherited Widgets (Flutter Community)](https://medium.com/flutter-community/inherited-widgets-bc3110821969)
