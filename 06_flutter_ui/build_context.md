# Что такое BuildContext и зачем он передается в метод `build`?

> `BuildContext` — это **адрес виджета в дереве элементов**: через него Flutter понимает, где именно сейчас строится UI, и дает доступ к данным «сверху» (тема, локаль, размеры, навигация). На практике `BuildContext` — это интерфейс над `Element`, поэтому его передают в `build`, чтобы виджет мог безопасно читать окружение и подписываться на изменения inherited-данных.

## Разбор

### Коротко: зачем `context` в `build`

```dart
@override
Widget build(BuildContext context) {
  final theme = Theme.of(context);
  return Text('Hello', style: theme.textTheme.bodyLarge);
}
```

`build` не только возвращает UI, но и должен знать **контекст построения**:
- какая тема действует в этой точке дерева;
- какие размеры экрана/клавиатуры;
- какие inherited-данные доступны выше по дереву;
- куда навигировать (`Navigator.of(context)`).

Без `context` виджет был бы «слепым» к окружению.

### Widget, Element, BuildContext — как связаны

| Сущность | Роль |
|----------|------|
| `Widget` | Иммутабельная конфигурация («чертеж») |
| `Element` | Живой узел в дереве, связывает виджет с местом в UI |
| `BuildContext` | Интерфейс доступа к `Element` (без прямой работы с ним) |
| `RenderObject` | Геометрия и отрисовка |

Когда виджет встраивается в дерево, вызывается `widget.createElement()`, создается `Element`, и в `build` передается именно его `BuildContext`.

**Своими словами:** виджет — это описание, `Element` — «экземпляр в конкретном месте дерева», `BuildContext` — ручка к этому месту.

### Что можно делать через `BuildContext`

- читать inherited-данные: `Theme.of(context)`, `MediaQuery.of(context)`, `Localizations.of(context)`;
- искать предков: `findAncestorStateOfType`, `findAncestorWidgetOfExactType`;
- навигировать: `Navigator.of(context)`, `ScaffoldMessenger.of(context)`;
- подписываться на изменения inherited-виджетов через `dependOnInheritedWidgetOfExactType` (внутри `of`-методов).

Важно: `of(context)` ищет данные **выше** текущего `context` в дереве. Если `InheritedWidget` объявлен ниже, данные не найдутся.

### Почему это интерфейс, а не сам `Element`

`BuildContext` специально ограничивает API: разработчик получает только нужные операции (поиск предков, зависимости, `mounted`), а не прямой доступ ко всей внутренней механике `Element`.  
Это защищает жизненный цикл и инкапсулирует работу фреймворка.

### Частые ошибки

1. **Использовать `context` не из той точки дерева** (частая ошибка с `InheritedWidget`).
2. **Хранить `context` надолго** и вызывать его после `await` без проверки `mounted`.
3. **Думать, что `context` = виджет** — это контекст места в дереве (`Element`), а не сам `Widget`.

### Как отвечать на собеседовании

`BuildContext` передают в `build`, потому что UI зависит не только от полей виджета, но и от окружения в дереве.  
`context` — это «где я в UI-дереве», а `Element` под капотом обеспечивает стабильную связь между конфигурацией (`Widget`) и отрисовкой (`RenderObject`).

## Что почитать

* [Элементы: первое погружение — Яндекс Образование](https://education.yandex.ru/handbook/flutter/article/elements)
* [BuildContext class (Flutter API)](https://api.flutter.dev/flutter/widgets/BuildContext-class.html)
* [Element class (Flutter API)](https://api.flutter.dev/flutter/widgets/Element-class.html)
* [What does BuildContext do in Flutter?](https://stackoverflow.com/questions/49100196/what-does-buildcontext-do-in-flutter)
