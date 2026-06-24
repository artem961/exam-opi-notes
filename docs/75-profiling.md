# Билет 75. Профилирование приложений. Основные подходы

## Ответ

**Профилирование** — сбор данных о поведении программы во время выполнения с целью определения, где тратится время и память.

### Два основных подхода

#### 1. Sampling (сэмплирование)

Профилировщик с заданной частотой (например, 100 Гц) «прерывает» процесс и записывает текущий стек вызовов.

```
Время →
[main→serviceA→queryDB]
[main→serviceA→queryDB]    ← queryDB появляется часто
[main→serviceA→queryDB]      → это горячее место
[main→serviceB→render]
[main→serviceA→queryDB]
```

**Преимущества:** низкий overhead (1–5%), пригоден для продакшна.  
**Недостатки:** статистический — пропускает редкие, но медленные вызовы.

#### 2. Instrumentation (инструментирование)

В каждую функцию вставляется код измерения. Профилировщик знает точное время каждого вызова.

```java
// Примерно то, что делает инструментирующий профилировщик:
long start = System.nanoTime();
queryDB();
long elapsed = System.nanoTime() - start;
recordTiming("queryDB", elapsed);
```

**Преимущества:** полная точность, точное число вызовов.  
**Недостатки:** overhead 50–500%, нельзя использовать в продакшне.

### Профилировщики по языку/платформе

| Платформа | Sampling | Instrumentation |
|-----------|----------|-----------------|
| **Java** | async-profiler, JFR | JProfiler, YourKit |
| **Linux** | `perf`, eBPF | `gprof` |
| **Python** | `py-spy` | `cProfile`, `line_profiler` |
| **Node.js** | V8 built-in profiler | `clinic.js` |
| **C/C++** | `perf`, Valgrind | `gprof`, Intel VTune |
| **Windows** | ETW, VTune | Visual Studio Profiler |

### Flame Graph — визуализация

```
Ось X = доля CPU (чем шире, тем больше времени)
Ось Y = глубина стека вызовов

          ┌─────────────────┐
          │  queryDB (55%)  │  ← горячая функция
          ├───────┬─────────┤
          │parseSQ│ execStmt│
          ├───────┴────┬────┤
          │ serviceA        │
          ├─────────────────┤
          │     main        │
          └─────────────────┘
```

Самые широкие плато сверху — узкие места.

---

## Подробно

### CPU Profiling

Отвечает на вопрос: «Какие функции потребляют больше всего процессорного времени?»

```bash
# Java: async-profiler
./profiler.sh -d 30 -f flamegraph.html PID

# Linux: perf
perf record -g -p PID sleep 30
perf report --stdio
perf script | FlameGraph/stackcollapse-perf.pl | FlameGraph/flamegraph.pl > flame.svg
```

### Memory Profiling

Отвечает на вопрос: «Какие объекты занимают память и где они создаются?»

```bash
# Java: heap dump
jmap -dump:live,format=b,file=heap.hprof PID
# Открыть в Eclipse Memory Analyzer (MAT) или VisualVM

# Python:
python -m memory_profiler script.py
tracemalloc  # встроен в Python 3.4+
```

Что ищем: объекты с большим `retained heap` (сколько памяти освободится при удалении объекта).

### Profiling vs Benchmarking

**Профилирование** — «где тратится время» (относительное сравнение).  
**Бенчмаркинг** — «сколько времени» (абсолютное измерение с точностью).

Для микробенчмарков Java используют JMH (Java Microbenchmark Harness) — учитывает warmup, компиляцию JIT, GC:

```java
@Benchmark
public void testMethod(Blackhole bh) {
    bh.consume(expensiveOperation());
}
```

### Профилирование в продакшне

Sampling profiler (async-profiler, perf) с overhead 1–5% можно включать в продакшне на короткое время (30 сек – 5 мин). Это безопаснее, чем воспроизводить проблему в тестовой среде с другой нагрузкой.

Continuous Profiling (например, Pyroscope, Grafana Phlare) — профилировщик работает постоянно, данные агрегируются. Позволяет найти регрессию производительности после деплоя.

### On-CPU vs Off-CPU Profiling

**On-CPU profiling** (обычный) — фиксирует стек только когда поток выполняется на CPU. Не покажет ожидание блокировки или I/O.

**Off-CPU profiling** — фиксирует стек когда поток заблокирован (ожидает mutex, I/O, sleep). Если программа «медленная» но CPU низкий — нужен off-CPU profiler.

```bash
# async-profiler: off-CPU profiling
./profiler.sh -e wall -d 30 PID  # wall clock: учитывает время ожидания
```
