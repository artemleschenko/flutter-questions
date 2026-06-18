# Как изменилась логика обработки системной кнопки «Назад» при переходе от `WillPopScope` к `PopScope`?

> Раньше back обрабатывали **в момент жеста** через `onWillPop` (можно было отменить pop). Для Android Predictive Back нужно знать заранее, разрешён ли pop, поэтому `WillPopScope` заменили на `PopScope` с `canPop` и `onPopInvoked`. Главное отличие: `onWillPop` вызывался до pop и мог его отменить; `onPopInvoked` — после обработки pop, а решение «можно ли» принимается заранее через `canPop`.

## Разбор

### Почему API поменяли

Android 14 добавил **Predictive Back**: анимация «заглянуть назад» начинается сразу при жесте, до подтверждения. Flutter больше не может в этот момент асинхронно решать, отменять pop или нет — ответ должен быть **известен заранее**.

| Модель | API | Когда решаем | Можно отменить pop? |
|--------|-----|--------------|---------------------|
| Just-in-time | `WillPopScope.onWillPop` | В момент back | Да (`return false`) |
| Ahead-of-time | `PopScope.canPop` | На каждом build | Нет; только `canPop: false` заранее |

### `WillPopScope` — как было

```dart
WillPopScope(
  onWillPop: () async {
    if (_hasUnsavedChanges) {
      return await _showDiscardDialog();
    }
    return true; // разрешить pop
  },
  child: child,
)
```

- `onWillPop` — `Future<bool>`, вызывается **до** pop.
- `true` → pop продолжается, `false` → отмена.
- Типичный сценарий: диалог «сохранить изменения?» при системном back.

### `PopScope` — как стало

```dart
PopScope(
  canPop: !_hasUnsavedChanges,
  onPopInvokedWithResult: (bool didPop, Object? result) {
    if (!didPop) {
      _showDiscardDialogAndMaybePop();
    }
  },
  child: child,
)
```

- `canPop` — можно ли pop **прямо сейчас** (обновляется при rebuild).
- `onPopInvoked` / `onPopInvokedWithResult` — callback **после** попытки pop.
- `didPop == true` → pop прошёл; `false` → pop заблокирован (`canPop: false`).

Диалог подтверждения при back:

```dart
PopScope(
  canPop: false,
  onPopInvoked: (didPop) async {
    if (didPop) return;

    final shouldPop = await _showBackDialog();
    if (shouldPop ?? false && context.mounted) {
      Navigator.of(context).pop();
    }
  },
  child: child,
)
```

### Сравнение поведения

| | `WillPopScope` | `PopScope` |
|---|----------------|------------|
| Решение «можно ли уйти» | `onWillPop` → `bool` | `canPop` |
| Момент callback | До pop | После pop |
| Отмена pop | `return false` | `canPop: false` |
| Async-логика в callback | Да | Только side effects; pop отдельно через `Navigator.pop()` |
| Predictive Back | Ломает/отключает | Поддерживает |

### Практические правила миграции

1. Блокировка back → `canPop: false`, не async в callback.
2. Побочный эффект при back → `onPopInvoked`, проверяй `didPop`.
3. Диалог подтверждения → `canPop: false` + в callback показать диалог + `Navigator.pop()` при согласии.
4. Разная логика для «системный back» и «кнопка в UI» → не полагайся только на `onPopInvoked`; раздели обработчики явно.
5. Для predictive back на Android: `android:enableOnBackInvokedCallback="true"` в `AndroidManifest.xml`, не использовать `WillPopScope`.

## Что почитать

* [Android predictive back — Flutter breaking changes](https://docs.flutter.dev/release/breaking-changes/android-predictive-back)
* [WillPopScope cannot be fully replaced with PopScope (issue #163052)](https://github.com/flutter/flutter/issues/163052)
* [So you thought you knew back navigation in Flutter](https://medium.com/write-a-catalyst/so-you-thought-you-knew-back-navigation-in-flutter-think-again-254ec753a6cf)
