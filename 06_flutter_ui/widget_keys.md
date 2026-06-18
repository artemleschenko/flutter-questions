# Для чего во Flutter нужны ключи (Key)? Приведи пример, когда без ValueKey сломается логика UI. В чем разница между GlobalKey и LocalKey?

> `Key` нужен, чтобы Flutter **правильно сопоставлял виджеты и `Element`/`State` при перестроении дерева** — особенно когда виджеты одного типа меняют порядок, удаляются или добавляются. Без ключа фреймворк часто сопоставляет по позиции (`runtimeType` + порядок), и состояние «переезжает» не туда. `LocalKey` работает среди соседей на одном уровне (`ValueKey`, `ObjectKey`, `UniqueKey`), а `GlobalKey` — глобально во всём дереве и даёт доступ к `State` извне.

## Разбор

### Зачем нужен `Key`

При rebuild Flutter сравнивает старое и новое дерево виджетов.  
Если тип и `key` совпали — переиспользуется тот же `Element` и тот же `State`.  
Если нет — `Element` может быть перемещён, пересоздан или удалён.

Это критично для `StatefulWidget`: `State` живёт дольше, чем экземпляр виджета.

### Пример: без `ValueKey` ломается UI

Сценарий: список задач, у каждой в `initState` задаётся случайный цвет фона.  
Удаляем элемент из середины — цвета «перепутались».

```dart
class TodoTile extends StatefulWidget {
  const TodoTile({required this.title, super.key});

  final String title;

  @override
  State<TodoTile> createState() => _TodoTileState();
}

class _TodoTileState extends State<TodoTile> {
  late final Color _bg = _randomColor();

  @override
  Widget build(BuildContext context) {
    return Container(
      color: _bg,
      child: Text(widget.title),
    );
  }
}
```

```dart
ListView(
  children: items
      .map((item) => TodoTile(title: item.title)) // key = null
      .toList(),
);
```

Было: `[A(red), B(blue), C(green)]` → удалили `A` → стало `[B, C]`.  
Flutter сопоставил по позиции:
- 1-й `Element` остался, но получил виджет `B` → цвет остался `red` (от старого `A`);
- 2-й `Element` получил `C` → цвет `blue` (от старого `B`).

**Итог:** данные обновились, а визуальное состояние — нет.

### Исправление через `ValueKey`

```dart
ListView(
  children: items
      .map(
        (item) => TodoTile(
          key: ValueKey(item.id), // стабильный id сущности
          title: item.title,
        ),
      )
      .toList(),
);
```

Теперь Flutter сопоставляет по `id`, а не по индексу: `State` переезжает вместе с правильной задачей.

### `GlobalKey` vs `LocalKey`

| Критерий | `LocalKey` | `GlobalKey` |
|----------|------------|-------------|
| Область уникальности | Среди соседей на одном уровне дерева | Во всём приложении |
| Основная задача | Сохранить `State` при reorder/add/remove в списке | Сохранить `State` при переносе в другое место дерева + доступ к `State` |
| Типы | `ValueKey`, `ObjectKey`, `UniqueKey` | `GlobalKey`, `GlobalObjectKey` |
| Стоимость | Обычно дешевле | Тяжелее, использовать осознанно |
| Типичные кейсы | `ListView` с `Stateful` элементами | `Form` (`validate()`), `Navigator`, widget tests |

**`LocalKey` — когда достаточно «узнать элемент среди соседей».**  
Классика: `ValueKey(item.id)` в списке.

**`GlobalKey` — когда нужен доступ к `State` или перенос виджета между ветками дерева.**

### Какой `LocalKey` выбрать

| Ключ | Когда использовать |
|------|--------------------|
| `ValueKey(id)` | Есть стабильный id/значение (`user.id`, `task.id`) |
| `ObjectKey(obj)` | Нужна идентификация по ссылке на объект |
| `UniqueKey()` | Нужен гарантированно новый элемент (осторожно: каждый rebuild создаст новый key, если писать прямо в `build`) |

### Частые ошибки

1. Ставить `UniqueKey()` в `build` для каждого элемента — постоянное пересоздание `State`.
2. Использовать `ValueKey(title)` при неуникальных title — дубликаты ключей.
3. Путать «ключ виджета» (`super.key` в конструкторе) и бизнес-id в `ValueKey(item.id)`.
4. Злоупотреблять `GlobalKey` там, где хватит `ValueKey` или обычного state management.

### Как отвечать на собеседовании

`Key` — это идентификатор для сопоставления виджетов при diff деревьев.  
Без `ValueKey` в динамическом списке `Stateful` элементов состояние часто «липнет» к позиции, а не к данным.  
`LocalKey` — для соседей, `GlobalKey` — глобально и для доступа к `State`.

## Что почитать

* [Ключи во Flutter — Habr/Surf](https://habr.com/ru/companies/surfstudio/articles/813939/)
* [Виджеты: идентификатор key — Яндекс Образование](https://education.yandex.ru/handbook/flutter/article/vidzheti-identifikator-key#chto-takoe-key)
* [Widget Keys (Flutter Widget of the Week)](https://www.youtube.com/watch?v=kn0EOS-ZiIc&t=2s)
* [Key class (Flutter API)](https://api.flutter.dev/flutter/foundation/Key-class.html)
* [GlobalKey class (Flutter API)](https://api.flutter.dev/flutter/widgets/GlobalKey-class.html)
