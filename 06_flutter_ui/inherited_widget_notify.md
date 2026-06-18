# Как InheritedWidget уведомляет своих потомков об изменениях? Почему `dependOnInheritedWidgetOfExactType<T>()` имеет сложность O(1), несмотря на потенциально глубокое дерево виджетов?

> Потомок **подписывается** на `InheritedWidget` через `context.dependOnInheritedWidgetOfExactType` (обычно внутри `of(context)`). Когда родитель пересоздаёт `InheritedWidget`, фреймворк вызывает `updateShouldNotify`; если вернул `true`, все зависимые потомки помечаются на rebuild. Поиск inherited-виджета — O(1): у каждого `Element` есть кеш ближайших `InheritedElement` по типу, а не линейный подъём по дереву.

## Разбор

### Цепочка уведомления (коротко)

```text
1) Потомок в build вызывает AppConfig.of(context)
2) of -> dependOnInheritedWidgetOfExactType -> регистрация зависимости
3) Родитель меняет данные и пересоздаёт InheritedWidget
4) Вызывается updateShouldNotify(oldWidget)
5) Если true -> зависимые Element помечаются dirty -> build у потомков
```

### Шаг 1: подписка потомка

Ключевой момент: не `find`, а **`depend`**.

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
  bool updateShouldNotify(AppConfig oldWidget) =>
      apiHost != oldWidget.apiHost;
}
```

`dependOnInheritedWidgetOfExactType`:
- находит ближайший `InheritedWidget` нужного типа **выше** по дереву;
- **регистрирует** текущий `Element` как зависимый;
- из-за этого потомок перестроится, когда inherited-данные изменятся.

Подписываться можно в `build` и `didChangeDependencies` (не в `initState`).

### Шаг 2: решение «нужно ли уведомлять»

Когда родитель пересобирается и создаёт новый `InheritedWidget`, вызывается:

```dart
@override
bool updateShouldNotify(AppConfig oldWidget) {
  return apiHost != oldWidget.apiHost;
}
```

- `true` → уведомить зависимых потомков;
- `false` → потомки не перестраиваются из-за этого inherited-виджета.

Важно: сравнивать нужно **данные внутри inherited-виджета**, а не просто ссылку на внешний объект, если она не меняется.

### Шаг 3: перестроение зависимых потомков

После `updateShouldNotify == true` Flutter помечает зависимые `Element` как «грязные» и вызывает их `build`.

Поэтому `Theme.of(context)`, `MediaQuery.of(context)` и ваш `AppConfig.of(context)` автоматически «подтягивают» актуальные данные.

### `of` vs `get` vs `find`

| Метод | Подписка | Rebuild при изменении | Где использовать |
|-------|----------|------------------------|------------------|
| `dependOnInheritedWidgetOfExactType` | Да | Да | UI, который должен реагировать на изменения |
| `getInheritedWidgetOfExactType` | Нет | Нет | Разовое чтение без подписки |
| `findAncestorStateOfType` | Нет (другой механизм) | Нет | Поиск `State` предка, обычно O(N) по дереву |

Для inherited-данных предпочтительнее `depend/getInherited...` (доступ за O(1) через map в `Element`).

Почему `dependOnInheritedWidgetOfExactType<T>()` имеет O(1), даже при глубоком дереве: Flutter не поднимается каждый раз вручную по всем предкам. У каждого `Element` есть `Map<Type, InheritedElement>` — кеш **ближайших** inherited-виджетов. Ключ — точный тип `T`, значение — один `InheritedElement` (самый близкий предок этого типа).

### Практические правила

1. В `updateShouldNotify` сравнивай только реально изменившиеся поля.
2. Не возвращай всегда `true`, если не хочешь лишних rebuild.
3. Для нескольких независимых значений лучше разделить на несколько inherited-виджетов (или использовать `InheritedNotifier`/`Provider`).
4. Если нужно читать inherited в `initState`, сделай это в `didChangeDependencies`.

### Частые ошибки

1. Вызывать `of(context)` из неправильного `BuildContext`.
2. Подписываться через `depend...` там, где нужен только read (`get...`).
3. Хранить в `InheritedWidget` mutable-объект и всегда возвращать `false` в `updateShouldNotify`.
4. Ожидать уведомления у потомков, которые никогда не вызывали `dependOnInheritedWidgetOfExactType`.

## Что почитать

* [InheritedWidget class (Flutter API)](https://api.flutter.dev/flutter/widgets/InheritedWidget-class.html)
* [IoC: управление зависимостями — Яндекс Образование](https://education.yandex.ru/handbook/flutter/article/ioc)
* [Inherited Widgets (Widget of the Week)](https://www.youtube.com/watch?v=og-vJqLzg2c)
* [Pragmatic State Management (InheritedWidget)](https://www.youtube.com/watch?v=Zbm3hjPjQMk)
