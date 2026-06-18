# Как сверстать список, в котором может быть 1000 элементов, чтобы приложение не зависло?

> Для больших списков нужна **ленивая отрисовка**: строить только видимые элементы (`ListView.builder` / `SliverList`), не создавать 1000 виджетов сразу, по возможности фиксировать высоту (`itemExtent`), избегать `shrinkWrap` и `SingleChildScrollView + Column`. Если данных очень много — добавляют **пагинацию** и подгружают порциями.

## Разбор

### Главная идея

`ListView(children: [...])` создаёт работу **для всех** элементов сразу — даже для тех, что далеко за экраном.  
Для 1000 элементов это лишняя память и лишние `build`.

`ListView.builder` строит элемент **по требованию** — только для видимой области (+ небольшой буфер).

| Подход | Когда ок | Проблема на 1000 элементов |
|--------|----------|-----------------------------|
| `ListView(children: ...)` | Мало элементов (десятки) | Создаёт все child сразу |
| `ListView.builder` | Длинные/бесконечные списки | Подходит |
| `SliverList` / `SliverList.builder` | Длинный список внутри `CustomScrollView` (header, tabs, несколько sliver-блоков) | Подходит (тоже ленивый) |
| `SingleChildScrollView + Column` | Короткий статичный контент | Весь список в памяти сразу |
| `ListView(shrinkWrap: true)` внутри другого скролла | Редкие кейсы | Пересчёт размеров всех элементов |

### Базовый правильный шаблон

```dart
class UsersPage extends StatelessWidget {
  const UsersPage({required this.users, super.key});

  final List<User> users; // может быть 1000+

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: users.length,
      itemExtent: 72, // если высота строки фиксирована — быстрее скролл
      itemBuilder: (context, index) {
        final user = users[index];
        return UserTile(user: user); // вынеси в отдельный виджет
      },
    );
  }
}
```

Почему это работает:
- `itemBuilder` вызывается только для видимых индексов;
- `itemExtent` помогает скролл-механике заранее знать размеры;
- `UserTile` изолирует логику одной строки.

### Что ещё ускоряет длинные списки

1. **`const` там, где можно** — не пересоздавать неизменяемые виджеты при rebuild родителя.
2. **Лёгкий `itemBuilder`** — без HTTP, тяжёлых вычислений и лишнего `setState` на весь экран.
3. **`ListView.separated`** вместо `Padding/Divider` на каждый элемент — меньше лишних виджетов.
4. **`RepaintBoundary`** для тяжёлых карточек (если профайлер показывает лишние перерисовки).
5. **`AutomaticKeepAliveClientMixin`** — только если реально нужно сохранять состояние элемента при скролле (иначе лишняя память).

### Если 1000 — это не «все сразу в памяти», а поток данных

Тогда добавляют **пагинацию / infinite scroll**:
- в state хранят уже загруженные элементы;
- при достижении конца списка догружают следующую страницу;
- показывают индикатор загрузки внизу.

Так UI остаётся отзывчивым: в памяти и на экране только нужная часть данных.

### Как отвечать на собеседовании

Коротко:
1. Использовать `ListView.builder` (ленивая сборка).
2. Не строить все 1000 элементов заранее.
3. Упростить элемент списка и ограничить область rebuild.
4. При необходимости — пагинация и фиксированная высота строки.
5. Проверить в DevTools (Performance/Timeline), а не «оптимизировать вслепую».

## Что почитать

* [ListView class (Flutter API)](https://api.flutter.dev/flutter/widgets/ListView-class.html)
* [Оптимизация Flutter-приложения: списки, build() — Habr/Friflex](https://habr.com/ru/companies/friflex/articles/1018320/)
* [Flutter Infinite List (BlocLibrary)](https://bloclibrary.dev/ru/tutorials/flutter-infinite-list/)
