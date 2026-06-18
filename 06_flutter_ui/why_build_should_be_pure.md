# Почему метод `build` должен быть «чистым»?

> `build` должен быть «чистым», потому что Flutter может вызывать его много раз и в неожиданные моменты: при изменении зависимостей, `setState`, hot reload/restart, изменении размеров экрана, навигации и перестроении родителей. Если в `build` есть сайд-эффекты (HTTP, запись в состояние, подписки, аналитика), они начнут выполняться повторно и ломать предсказуемость UI.

## Разбор

### Что значит «чистый build»

«Чистый» `build`:
- получает текущие входные данные (`widget` + `context`);
- возвращает дерево виджетов;
- **не** меняет состояние и **не** запускает побочные действия.

Практическая формула:
- `build = description of UI`
- `side effects = lifecycle/callbacks`, но не `build`.

### Почему Flutter так требует

У `StatelessWidget.build` в официальной документации прямо сказано: метод потенциально может вызываться хоть каждый кадр и не должен иметь сайд-эффектов, кроме построения виджетов.  

Это архитектурная основа декларативного UI: фреймворк свободно решает, когда пересчитать описание интерфейса, а разработчик не завязывает на этот пересчет сетевые вызовы и мутации.

### Что обычно триггерит повторный `build`

- `setState()` в текущем `State`;
- перестроение родителя;
- изменение зависимостей из `context` (например, `Theme`, `MediaQuery`, локализация);
- изменение размеров/ориентации, показ клавиатуры;
- hot reload/hot restart.

Поэтому «`build` вызывается несколько раз» — это нормальное поведение, а не баг само по себе.

### Что ломается, если `build` не чистый

Самый частый антипаттерн:

```dart
@override
Widget build(BuildContext context) {
  return FutureBuilder(
    future: httpCall(), // плохо: каждый новый build может стартовать новый запрос
    builder: (context, snapshot) => ...
  );
}
```

Проблемы:
- дублирующиеся HTTP-запросы;
- мигающий UI (`waiting` -> `done` повторно);
- лишняя нагрузка и нестабильные тесты;
- трудно отлаживать, потому что «что-то само повторно вызвалось».

### Как правильно

Идея: **инициализировать один раз, переиспользовать в build**.

```dart
class Example extends StatefulWidget {
  const Example({super.key});

  @override
  State<Example> createState() => _ExampleState();
}

class _ExampleState extends State<Example> {
  late Future<Data> _future;

  @override
  void initState() {
    super.initState();
    _future = httpCall(); // один запуск
  }

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<Data>(
      future: _future,
      builder: (context, snapshot) {
        // только рендер
        return const SizedBox.shrink();
      },
    );
  }
}
```

Если нужен явный рефреш — делай его через событие (`onPressed`) и `setState`, где обновляешь `_future`.

### Где тогда делать сайд-эффекты

- одноразовая инициализация: `initState`;
- реакция на смену конфигурации: `didUpdateWidget`;
- реакция на inherited-зависимости: `didChangeDependencies`;
- освобождение ресурсов: `dispose`;
- пользовательские события: `onPressed`, `onTap`, listeners.

### Частые ошибки

1. Создавать `Future`/`Stream` прямо в `build`.
2. Логировать аналитику в `build` как «событие показа».
3. Пытаться «запретить rebuild» вместо того, чтобы сделать `build` безопасным.
4. Путать оптимизацию rebuild и корректность: сначала чистый `build`, потом точечные оптимизации (`const`, вынос поддеревьев, `ValueListenableBuilder` и т.д.).

## Что почитать

* [build method - StatelessWidget class](https://api.flutter.dev/flutter/widgets/StatelessWidget/build.html)
* [How to deal with unwanted widget build?](https://stackoverflow.com/questions/52249578/how-to-deal-with-unwanted-widget-build)
* [Why build function gets called twice when hot restarting?](https://stackoverflow.com/questions/64834702/why-build-function-gets-called-twice-when-hot-restarting)
* [Flutter: StatelessWidget.build called multiple times](https://stackoverflow.com/questions/53223469/flutter-statelesswidget-build-called-multiple-times)
