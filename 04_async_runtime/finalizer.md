# Для каких задач предназначены Finalizer и почему их нельзя использовать как замену dispose()?

> `Finalizer<T>` — механизм, позволяющий зарегистрировать callback, который [GC](garbage_collector.md) вызовет **когда-нибудь после** того, как прикреплённый объект станет недостижимым. Предназначен для освобождения **внешних ресурсов** (native-память через FFI, файловые дескрипторы, GPU-текстуры). Нельзя использовать как замену `dispose()`, потому что: (1) **нет гарантии времени вызова** — GC может собрать объект через секунду или никогда; (2) callback получает только «токен» (detach key), **не сам объект**; (3) ресурсы (анимации, подписки, контроллеры) во Flutter должны освобождаться **детерминированно** в `dispose()`, а не «когда-нибудь».

## Разбор

### API Finalizer

```dart
// 1. Создаём Finalizer с callback
final finalizer = Finalizer<int>((pointer) {
  // Этот код выполнится КОГДА-НИБУДЬ после того,
  // как объект, к которому привязан pointer, станет недостижимым
  nativeLib.free(pointer);
  print('Native memory at $pointer freed');
});

// 2. Привязываем объект
void useNativeResource() {
  final buffer = NativeBuffer(size: 1024);
  final nativePointer = buffer.pointer; // int — «токен» для cleanup

  finalizer.attach(
    buffer,         // объект, за которым следим
    nativePointer,  // токен для callback (НЕ сам объект!)
    detach: buffer, // ключ для ручной отвязки
  );

  // Если buffer станет недостижимым → GC вызовет callback с nativePointer
}

// 3. Ручная отвязка (если ресурс освобождён раньше)
void disposeManually(NativeBuffer buffer) {
  nativeLib.free(buffer.pointer);
  finalizer.detach(buffer); // Отменяем финализацию — ресурс уже освобождён
}
```

### Почему Finalizer ≠ dispose()

| Аспект | `dispose()` | `Finalizer` |
|--------|-------------|-------------|
| Когда вызывается | **Детерминированно** — при удалении из дерева | **Недетерминированно** — при следующем GC (может не случиться) |
| Кто вызывает | Framework (`State.dispose`, `ChangeNotifier.dispose`) | GC (вне вашего контроля) |
| Доступ к объекту | Полный (`this`) | **Нет** — только токен |
| Порядок вызовов | Гарантирован (дети → родители) | Не гарантирован |
| Для чего | Подписки, контроллеры, анимации, фокусы | Native-ресурсы (FFI) как **страховка** |

### Критический пример: почему нельзя заменить dispose

```dart
// ⚠️ НЕПРАВИЛЬНО: пытаемся освободить AnimationController через Finalizer
class _BadState extends State<BadWidget> {
  late final AnimationController _ctrl;
  static final _finalizer = Finalizer<AnimationController>((ctrl) {
    ctrl.dispose(); // 💥 Проблемы:
    // 1. Может вызваться через минуты → анимация «зомби» всё это время
    // 2. Может НЕ вызваться вообще (приложение закрылось раньше GC)
    // 3. Вызывается в непредсказуемый момент → possible use-after-dispose
  });

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(vsync: this);
    // ⚠️ Нельзя передать ctrl как token — callback получает КОПИЮ ссылки,
    // и это удерживает ctrl от GC → финализатор НИКОГДА не сработает!
  }
}

// ✅ ПРАВИЛЬНО: детерминированный dispose
class _GoodState extends State<GoodWidget> {
  late final AnimationController _ctrl;

  @override
  void initState() {
    super.initState();
    _ctrl = AnimationController(vsync: this, duration: Durations.medium2);
  }

  @override
  void dispose() {
    _ctrl.dispose(); // Гарантированно, немедленно, предсказуемо
    super.dispose();
  }
}
```

### Правильный use-case: FFI Native Resources

```dart
class NativeImage {
  final Pointer<Uint8> _data;
  final int _size;

  static final _finalizer = Finalizer<Pointer<Uint8>>((ptr) {
    // Safety net: если кто-то забыл вызвать close()
    calloc.free(ptr);
  });

  NativeImage(this._size)
      : _data = calloc<Uint8>(_size) {
    _finalizer.attach(this, _data, detach: this);
  }

  /// Детерминированное освобождение — основной путь
  void close() {
    calloc.free(_data);
    _finalizer.detach(this); // ← Отменяем финализатор
  }
}
```

Здесь `Finalizer` — **страховочная сеть**: если программист забыл `close()`, GC всё равно освободит native-память. Но основной путь — `close()`.

### NativeFinalizer — синхронный вариант для FFI

Для Dart FFI есть специализированный `NativeFinalizer`, который вызывает C-функцию `free` напрямую (без Dart callback):

```dart
final nativeFinalizer = NativeFinalizer(
  DynamicLibrary.open('mylib.so').lookup('my_free_function'),
);

class NativeTexture implements Finalizable {
  final Pointer<NativeType> _ptr;

  NativeTexture(this._ptr) {
    nativeFinalizer.attach(this, _ptr.cast(), detach: this);
  }

  void dispose() {
    myLib.freeTexture(_ptr);
    nativeFinalizer.detach(this);
  }
}
```

### Правила использования

1. **Токен НЕ должен ссылаться на объект.** Иначе объект никогда не соберётся → финализатор никогда не сработает.
2. **Всегда предоставляйте `detach`.** Чтобы при ручном `dispose()` / `close()` отменить финализатор.
3. **Finalizer — страховка, не основной путь.** Основной путь — детерминированный `dispose()` / `close()`.
4. **Не делайте в callback ничего тяжёлого** — он запускается GC, блокирует сборку.

## Что почитать

* [Dart: Finalizer class](https://api.dart.dev/stable/dart-core/Finalizer-class.html)
* [Dart: NativeFinalizer class](https://api.dart.dev/stable/dart-ffi/NativeFinalizer-class.html)
* [Dart FFI: Memory management](https://dart.dev/interop/c-interop#memory-management)
