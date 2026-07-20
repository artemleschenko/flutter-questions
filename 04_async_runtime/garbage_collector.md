# Как работает сборщик мусора (Garbage Collector)? Чем поколения (Generations) в Dart GC отличаются от Java?

> Dart использует **поколенческий (generational) GC** с двумя основными пространствами: **Young Space** (молодое поколение, semi-space copying collector) и **Old Space** (старое поколение, mark-sweep/mark-compact). Главное отличие от Java: Dart GC оптимизирован под **однопоточные Isolate** — нет stop-the-world пауз, разделяемых между потоками, и нет сложной синхронизации. Java GC (G1, ZGC, Shenandoah) решает проблему shared heap между потоками, что добавляет write-barriers и safepoints. Dart же выигрывает от изоляции: каждый [Isolate](../01_cs/multithreading_vs_async.md) имеет собственный heap, и GC работает только внутри него.

## Разбор

### Архитектура памяти Dart VM

```
┌──────────────────── Isolate Heap ────────────────────┐
│                                                       │
│  ┌─────────────────────────────────────┐              │
│  │         Young Space (New Gen)       │              │
│  │  ┌──────────┐   ┌──────────┐       │              │
│  │  │ From-space│   │ To-space │       │              │
│  │  │ (active)  │   │ (пустой) │       │              │
│  │  └──────────┘   └──────────┘       │              │
│  └─────────────────────────────────────┘              │
│                                                       │
│  ┌─────────────────────────────────────┐              │
│  │          Old Space (Old Gen)        │              │
│  │    Mark-Sweep + Mark-Compact        │              │
│  └─────────────────────────────────────┘              │
│                                                       │
│  ┌─────────────────────────────────────┐              │
│  │         Code Space (JIT/AOT)        │              │
│  └─────────────────────────────────────┘              │
└───────────────────────────────────────────────────────┘
```

### Young Space — Semi-Space Copying (Scavenge)

Новые объекты создаются в **Young Space** (обычно ~16 MB, настраивается). Пространство делится на два полупространства:

1. **From-space** — здесь живут объекты.
2. **To-space** — пустой, ждёт сборки.

При сборке (scavenge):
1. GC обходит корни (стек, глобалы, регистры).
2. Живые объекты **копируются** из From-space в To-space.
3. From и To меняются ролями.
4. Мёртвые объекты не удаляются — их просто забывают (всё From-space сбрасывается).

```dart
void example() {
  // Все эти объекты создаются в Young Space
  final list = List.generate(1000, (i) => Point(i, i * 2));
  final filtered = list.where((p) => p.x > 500).toList();
  // list — если больше не используется, не попадёт в To-space → «умрёт»
  // filtered — живой, скопируется в To-space
}
```

**Скорость:** O(живые объекты), а не O(все объекты). Если большинство объектов короткоживущие (а во Flutter это так — виджеты пересоздаются каждый кадр), scavenge невероятно быстр.

### Промоция в Old Space

Объект, переживший **несколько scavenge-циклов** (обычно 2-3), «промотируется» в Old Space — предполагается, что он долгоживущий.

### Old Space — Mark-Sweep / Mark-Compact

Для старого поколения Dart VM использует:

1. **Mark** — обход графа объектов от корней, пометка живых.
2. **Sweep** — сбор мёртвых объектов, возврат памяти в free-list.
3. **Compact** (периодически) — дефрагментация: живые объекты сдвигаются в начало, указатели обновляются.

Old Space GC запускается реже и может быть **инкрементальным** — работа разбивается на мелкие шаги между кадрами, чтобы не блокировать [Event Loop](event_loop.md).

### Сравнение с Java GC

| Аспект | Dart GC | Java GC (G1/ZGC) |
|--------|---------|-------------------|
| Модель памяти | Изолированный heap на Isolate | Shared heap между Thread'ами |
| Поколения | 2 (Young + Old) | 3+ (Young: Eden + 2 Survivor, Old, Metaspace) |
| Young GC | Semi-space copy (scavenge) | Copying collector (Eden → Survivor) |
| Old GC | Incremental mark-sweep-compact | G1: region-based concurrent mark, ZGC: colored pointers |
| Stop-the-world | Минимальный (только scavenge ~ms) | G1: yes (minor pauses), ZGC: sub-ms pauses |
| Write barrier | Простой (нет shared state) | Сложный (card table, SATB, remembered sets) |
| Thread safety | Не нужна (Isolate = single-thread) | CAS, safepoints, memory fences |
| Настройка | `--old_gen_heap_size`, `--new_gen_semi_max_size` | Десятки флагов (`-Xms`, `-Xmx`, `-XX:G1HeapRegionSize` и т.д.) |
| Finalization | [`Finalizer`](finalizer.md) (weak + callback) | `finalize()` (deprecated), `Cleaner` |

### Почему Flutter-приложения выигрывают

1. **Виджеты — короткоживущие.** При `setState` дерево виджетов пересоздаётся — это идеальный сценарий для Young Space scavenge.
2. **Нет shared heap.** Каждый Isolate собирает мусор независимо — нет координации с другими потоками.
3. **`const` виджеты** — канонизированные объекты вообще не создаются повторно и не нагружают GC.
4. **Инкрементальный Old GC** разбивает работу между кадрами — пользователь не замечает пауз.

### Практические советы

```dart
// ✅ const виджеты — не создают новых объектов, GC их не трогает
const SizedBox(height: 16);
const Padding(padding: EdgeInsets.all(8));

// ⚠️ Утечка: подписка на Stream без cancel
@override
void initState() {
  super.initState();
  _subscription = stream.listen((_) => setState(() {}));
}

@override
void dispose() {
  _subscription.cancel(); // ✅ Обязательно — иначе объект не соберётся
  super.dispose();
}
```

## Что почитать

* [Dart: Memory overview](https://docs.flutter.dev/tools/devtools/memory)
* [Vyacheslav Egorov: Dart VM Internals (Google I/O)](https://mrale.ph/)
* [Flutter: Performance best practices](https://docs.flutter.dev/perf/best-practices)
