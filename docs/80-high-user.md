<div style="background:#d32f2f;color:#fff;padding:1.3rem 1.5rem;border-radius:8px;font-size:1.5rem;font-weight:800;line-height:1.35;text-align:center;margin:0 0 1.6rem 0;box-shadow:0 2px 8px rgba(0,0,0,.25)">
Полина Матвеева может не готовиться, всё равно она не сдаст ОПИ завтра.
</div>

# Билет 80. Рецепты повышения производительности при высоком %User

## Ответ

**%User (user time)** — доля процессорного времени, затраченная на выполнение кода самого приложения (не ядра).

Высокий %user означает, что приложение — настоящий потребитель CPU. Решение находится в коде приложения.

![Рецепты при высоком %user: уменьшить алгоритмическую сложность, переиспользование объектов, убрать опросы/ожидания в циклах, промахи кэша и TLB, организация кэшей, распараллеливание](assets/lec-recipes-high-user.png)

### Диагностика

```bash
# 1. Подтвердить: %user высокий
vmstat 1        # колонка us
top             # %CPU по процессам

# 2. Найти горячую функцию
perf record -g -p PID sleep 30
perf report

# Java: async-profiler
./profiler.sh -d 30 -f flamegraph.html PID
# открыть flamegraph.html — широкие блоки сверху = горячие функции

# Python: py-spy
py-spy top --pid PID
```

### Причины и рецепты

| Причина | Рецепт |
|---------|--------|
| **Неэффективный алгоритм** O(n²) | Заменить алгоритмом O(n log n) или O(n) |
| **Отсутствие кэша** | Мемоизация, кэширование результатов |
| **Избыточная сериализация** | Использовать быстрый формат (Protobuf, MessagePack) |
| **Неэффективная работа со строками** | StringBuilder вместо конкатенации, пулы строк |
| **Регулярные выражения в цикле** | Компилировать Pattern заранее |
| **Лишние объекты в цикле** | Переиспользовать объекты (object pool) |
| **Избыточное логирование** | Проверять `isDebugEnabled()`, async logging |

### Рецепт 1: Улучшение алгоритма

```java
// O(n²): проверить, есть ли дубликаты в массиве
boolean hasDuplicate = false;
for (int i = 0; i < n; i++)
    for (int j = i+1; j < n; j++)
        if (a[i] == a[j]) hasDuplicate = true;
// При n=10 000 → 100 000 000 итераций

// O(n): то же самое через HashSet
boolean hasDuplicate = new HashSet<>(Arrays.asList(a)).size() < n;
// При n=10 000 → 10 000 итераций
```

### Рецепт 2: Кэширование в коде (мемоизация)

```java
// Без кэша: каждый вызов — тяжёлое вычисление
double result = expensiveCalc(input);

// С кэшем: одно вычисление, потом из памяти
Map<Integer, Double> cache = new ConcurrentHashMap<>();
double result = cache.computeIfAbsent(input, this::expensiveCalc);
```

### Рецепт 3: Эффективная работа со строками

```java
// Медленно: каждая конкатенация создаёт новый объект String
String s = "";
for (String part : parts) s += part;  // O(n²)

// Быстро: StringBuilder переиспользует буфер
StringBuilder sb = new StringBuilder();
for (String part : parts) sb.append(part);  // O(n)
String s = sb.toString();
```

### Рецепт 4: Быстрая сериализация

```java
// JSON (Jackson): медленная сериализация
byte[] json = objectMapper.writeValueAsBytes(obj);

// Protobuf: компактнее, быстрее (3–10×)
byte[] proto = obj.toProtobuf().toByteArray();
```

---

## Подробно

### Flame Graph: как читать

```
         ┌───────────────────────────────────────┐
         │         processOrder (45%)            │  ← 45% CPU здесь!
         ├────────────────────┬──────────────────┤
         │   validateInput    │    saveOrder     │
         │       (5%)         │      (40%)       │
         ├────────────────────┼──────────────────┤
         │                    │  serialize (30%) │  ← 30% на сериализацию!
         │                    ├──────────────────┤
         │                    │  dbWrite (10%)   │
         └────────────────────┴──────────────────┘
                 main
```

Вывод: оптимизировать `serialize` — это 30% от всего времени.

### Преждевременная оптимизация

Не оптимизировать без данных. Сначала измерить, потом оптимизировать. Оптимизация «на глаз» часто улучшает не то место.

Цикл оптимизации:
```
Измерить → Найти узкое место → Оптимизировать → Измерить снова
```

### Регулярные выражения — скрытый источник CPU

```java
// Медленно: Pattern.compile() при каждом вызове метода
boolean matches = str.matches("\\d{3}-\\d{4}");  // компилирует каждый раз

// Быстро: компилировать один раз
private static final Pattern PHONE = Pattern.compile("\\d{3}-\\d{4}");
boolean matches = PHONE.matcher(str).matches();
```

### JIT-компиляция и inline

JVM автоматически встраивает (inline) часто вызываемые мелкие методы. Небольшие методы (< 35 байт байткода) встраиваются всегда — не нужно вручную «инлайнить» код.

```bash
# Посмотреть, что JIT скомпилировал
java -XX:+PrintCompilation -jar app.jar 2>&1 | grep "made not entrant"
```

### Избыточное создание объектов

В Java каждый `new Object()` добавляет давление на GC. В высоконагруженных местах:

```java
// Избегать создания объектов в горячем пути
// Плохо: создаём список на каждый запрос
List<Item> result = new ArrayList<>();

// Лучше: пул объектов или ThreadLocal
private static final ThreadLocal<List<Item>> listPool =
    ThreadLocal.withInitial(ArrayList::new);

List<Item> result = listPool.get();
result.clear();  // переиспользовать
```

### Параллелизм при высоком %user

Если однопоточный код загружает одно ядро на 100%, а остальные простаивают — распараллелить:

```java
// Многоядерный параллелизм (Java 8+)
long count = bigList.parallelStream()
    .filter(predicate)
    .count();
```

Но: синхронизация и overhead параллелизма окупаются только при достаточно большом объёме работы. Для малых объёмов — последовательный код быстрее.
