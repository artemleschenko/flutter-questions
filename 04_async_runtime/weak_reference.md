# Когда объект должен удерживаться через WeakReference, а не через обычную ссылку?

> `WeakReference<T>` удерживает объект **без предотвращения его сборки** [GC](garbage_collector.md). Если на объект не осталось обычных (сильных) ссылок, GC соберёт его, а `weakRef.target` вернёт `null`. Используйте `WeakReference`, когда вы хотите **иметь доступ к объекту, пока он жив, но не хотите быть причиной его выживания** — кэши, observer-реестры, обратные ссылки в графах.

## Разбор

### API

```dart
import 'dart:core';

void example() {
  var heavy = HeavyObject();
  final weakRef = WeakReference(heavy);

  print(weakRef.target); // HeavyObject (жив — есть сильная ссылка heavy)

  heavy = HeavyObject(); // старый объект потерял последнюю сильную ссылку

  // После GC:
  // weakRef.target == null (объект собран)
}
```

### Когда использовать

#### 1. Кэш с автоматическим вытеснением

Кэш не должен мешать GC собирать объекты, которые никто больше не использует:

```dart
class ImageCache {
  final _cache = <String, WeakReference<Image>>{};

  Image? get(String key) {
    final ref = _cache[key];
    if (ref == null) return null;

    final image = ref.target;
    if (image == null) {
      _cache.remove(key); // Объект был собран — чистим запись
      return null;
    }
    return image;
  }

  void put(String key, Image image) {
    _cache[key] = WeakReference(image);
  }
}
```

> Без `WeakReference` кэш удерживал бы все изображения в памяти вечно → утечка на устройствах с ограниченной RAM.

#### 2. Observer / Listener Registry

Глобальная шина событий не должна предотвращать GC виджетов:

```dart
class EventBus {
  final _listeners = <WeakReference<EventListener>>[];

  void register(EventListener listener) {
    _listeners.add(WeakReference(listener));
  }

  void emit(Event event) {
    _listeners.removeWhere((ref) => ref.target == null); // чистка мёртвых
    for (final ref in _listeners) {
      ref.target?.onEvent(event);
    }
  }
}
```

#### 3. Обратные ссылки в графе

Дочерний узел хочет «знать» о родителе, но не удерживать его:

```dart
class TreeNode {
  final List<TreeNode> children = [];
  WeakReference<TreeNode>? _parentRef;  // не удерживает родителя

  TreeNode? get parent => _parentRef?.target;

  void addChild(TreeNode child) {
    children.add(child);
    child._parentRef = WeakReference(this);
  }
}
```

### Когда НЕ использовать

| Сценарий | Почему не `WeakReference` |
|----------|--------------------------|
| DI-зависимости | Сервис должен жить, пока нужен. Сильная ссылка — единственная гарантия |
| State management | BLoC/Cubit удерживается через `BlocProvider`, нужна гарантия жизни |
| Подписки на Stream | Используйте `cancel()` в `dispose()`, а не слабые ссылки |
| `final` поля модели | Модель должна полностью владеть своими данными |

### WeakReference + Finalizer

Типичный паттерн: `WeakReference` для доступа, [`Finalizer`](finalizer.md) для очистки при сборке:

```dart
class NativeResourceManager {
  static final _pointers = <int, WeakReference<NativeHandle>>{};
  static final _finalizer = Finalizer<int>((pointer) {
    // Вызывается GC, когда NativeHandle собран
    ffi.freeNativeMemory(pointer);
    _pointers.remove(pointer);
  });

  NativeHandle allocate(int size) {
    final pointer = ffi.allocateNativeMemory(size);
    final handle = NativeHandle(pointer);
    _pointers[pointer] = WeakReference(handle);
    _finalizer.attach(handle, pointer);
    return handle;
  }
}
```

### Expando — похожий механизм

`Expando<T>` — встроенный в Dart «слабый словарь» (weak map): ключи хранятся слабо, значения удаляются при сборке ключа.

```dart
final _metadata = Expando<Metadata>('widget metadata');

void attachMeta(Widget widget, Metadata meta) {
  _metadata[widget] = meta; // Когда widget собран GC → meta тоже удалится
}

Metadata? getMeta(Widget widget) => _metadata[widget];
```

## Что почитать

* [Dart: WeakReference class](https://api.dart.dev/stable/dart-core/WeakReference-class.html)
* [Dart: Expando class](https://api.dart.dev/stable/dart-core/Expando-class.html)
* [Dart: Weak references and finalizers](https://dart.dev/language/memory)
