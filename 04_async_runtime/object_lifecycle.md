# Каков жизненный цикл объекта в Dart с момента выделения памяти в Young Space до его удаления?

> Объект рождается в **Young Space** (bump-pointer allocation — мгновенно), переживает 1-2 [scavenge-цикла](garbage_collector.md) (semi-space copy), **промотируется** в **Old Space** если живёт достаточно долго, и в конечном итоге помечается как мёртвый в фазе **mark-sweep**. Если к объекту привязан [`Finalizer`](finalizer.md), GC вызовет callback перед освобождением памяти. Весь цикл спроектирован под Flutter-реальность: большинство объектов (виджеты, промежуточные коллекции) умирают молодыми, и scavenge собирает их за микросекунды.

## Разбор

### Полный жизненный цикл

```
                        ┌─────────────────────────────────────────────────────┐
                        │               РОЖДЕНИЕ (Allocation)                 │
                        │  Bump-pointer в Young Space From-space              │
                        │  Стоимость: O(1) — просто сдвиг указателя          │
                        └────────────────────┬────────────────────────────────┘
                                             │
                                             ▼
                        ┌─────────────────────────────────────────────────────┐
                        │              ЖИЗНЬ В YOUNG SPACE                    │
                        │  Объект доступен через стек / поля других объектов  │
                        └────────────────────┬────────────────────────────────┘
                                             │
                                     ┌───────┴───────┐
                                     ▼               ▼
                              Объект достижим?    Объект НЕ достижим
                                     │               │
                                     ▼               ▼
                        ┌─────────────────┐  ┌────────────────────┐
                        │ SCAVENGE (Copy) │  │  НЕ копируется     │
                        │ From → To-space │  │  From-space обнулён │
                        └────────┬────────┘  │  → ПАМЯТЬ СВОБОДНА │
                                 │           └────────────────────┘
                                 ▼
                        Пережил 2-3 scavenge?
                        ┌───────┴────────┐
                        нет              да
                        │                │
                        ▼                ▼
                   Остаётся в      ┌──────────────────┐
                   Young Space     │ ПРОМОЦИЯ в       │
                   (следующий       │ Old Space         │
                    scavenge)      └────────┬─────────┘
                                           │
                                           ▼
                        ┌─────────────────────────────────────────────────────┐
                        │              ЖИЗНЬ В OLD SPACE                      │
                        │  Incremental Mark-Sweep (между кадрами)             │
                        └────────────────────┬────────────────────────────────┘
                                             │
                                     ┌───────┴───────┐
                                     ▼               ▼
                              Достижим (mark)   НЕ достижим (sweep)
                                     │               │
                                     ▼               ▼
                               Живёт дальше   ┌────────────────────┐
                                              │ Есть Finalizer?    │
                                              │ Да → callback      │
                                              │ Нет → free          │
                                              └────────────────────┘
```

### Фаза 1: Allocation (рождение)

```dart
// Каждый new / literal создаёт объект в Young Space
final point = Point(3, 4);       // bump-pointer: O(1)
final list = [1, 2, 3];          // тоже Young Space
final map = {'key': 'value'};    // и это тоже
```

Dart VM использует **bump-pointer allocator**: есть указатель на «следующее свободное место» в From-space, аллокация — просто сдвиг этого указателя на размер объекта. Это быстрее `malloc` в C.

### Фаза 2: Scavenge (Young GC)

Запускается, когда From-space заполнен (обычно ~8-16 MB). GC обходит корни → копирует живые объекты в To-space → From и To меняются ролями.

```dart
void buildFrame() {
  // Каждый кадр Flutter создаёт ~сотни виджетов
  final widget = Container(
    padding: const EdgeInsets.all(8),  // const → не в Young Space (канонизирован)
    child: Text('Hello'),             // Text — в Young Space
  );
  // После build: widget, Text — если не удержаны Element'ом, умрут при scavenge
}
```

**Почему это быстро:** стоимость scavenge = O(живые объекты). Если 95% объектов мертвы (а во Flutter так и есть), копировать нужно только 5%.

### Фаза 3: Промоция

Объект, переживший N scavenge-циклов (обычно 2), копируется в Old Space. Примеры долгоживущих объектов:

```dart
// Эти объекты промотируются в Old Space:
class AppState {
  final UserRepository userRepo;     // живёт весь жизненный цикл приложения
  final SupabaseClient supabase;     // singleton
  final List<Incident> cachedItems;  // кэш данных
}
```

### Фаза 4: Old Space GC (Mark-Sweep-Compact)

Запускается реже, но обрабатывает больший объём. Dart VM делает это **инкрементально**:

1. **Mark** (инкрементально): обход графа от корней, пометка живых. Разбивается на маленькие шаги между кадрами.
2. **Sweep**: проход по Old Space, добавление мёртвых блоков в free-list.
3. **Compact** (периодически): сдвиг живых объектов → устранение фрагментации.

```
Кадр 1: [build] [mark 10%] [paint]
Кадр 2: [build] [mark 20%] [paint]
...
Кадр 10: [build] [mark 100%] [sweep] [paint]
```

### Фаза 5: Финализация и освобождение

```dart
class NativeTexture {
  final Pointer<Void> _ptr;
  static final _fin = Finalizer<Pointer<Void>>((ptr) => calloc.free(ptr));

  NativeTexture(int size) : _ptr = calloc<Void>(size) {
    _fin.attach(this, _ptr, detach: this);
  }

  // Жизненный цикл:
  // 1. new NativeTexture → Young Space (Dart) + native heap (C)
  // 2. Scavenge → скопирован в To-space (указатель _ptr обновлён)
  // 3. Промоция → Old Space
  // 4. Стал недостижимым → GC вызывает _fin callback → calloc.free(_ptr)
  // 5. Dart-память объекта возвращена в free-list
}
```

### Практический timeline (приблизительный)

| Событие | Время | Задержка UI |
|---------|-------|-------------|
| Allocation (bump-pointer) | ~нс | 0 |
| Scavenge (Young GC) | ~1-3 мс | Минимальная |
| Промоция | Во время scavenge | Включена в scavenge |
| Old GC mark (инкрементальный) | ~0.5 мс за шаг | Минимальная |
| Old GC sweep | ~1-2 мс | Минимальная |
| Old GC compact | ~5-10 мс | Заметная (редко) |

### Оптимизация для Flutter-разработчика

```dart
// ✅ const — объект не аллоцируется в Young Space вообще
const SizedBox(height: 16);

// ✅ Переиспользование объектов — меньше аллокаций
static final _borderRadius = BorderRadius.circular(12);

// ⚠️ Создание объектов в build() — нагружает Young Space
@override
Widget build(BuildContext context) {
  return Container(
    decoration: BoxDecoration(         // Новый объект каждый build!
      borderRadius: BorderRadius.circular(12), // И этот тоже!
    ),
  );
}
```

## Что почитать

* [Dart: Understanding memory allocation](https://docs.flutter.dev/tools/devtools/memory)
* [Dart VM: Garbage Collection internals](https://github.com/dart-lang/sdk/wiki/Garbage-Collection)
* [Flutter: Performance — rendering](https://docs.flutter.dev/perf/rendering-performance)
