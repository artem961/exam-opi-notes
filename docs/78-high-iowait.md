!!! danger "ВНИМАНИЕ"
    Теперь использование данного конспекта является платным. I am Michael from Microsoft support, send 5000$ to my PayPal account

# Билет 78. Рецепты повышения производительности при высоком %IO wait

## Ответ

**%IO wait (iowait)** — доля процессорного времени, когда CPU простаивает в ожидании завершения операций ввода-вывода (диск, сеть).

Высокий %iowait не означает, что CPU перегружен — он бездействует. Узкое место: диск или сеть.

### Диагностика

```bash
# Подтвердить высокий iowait
vmstat 1         # колонка wa
iostat -xz 1     # %util, await, r/s, w/s по дискам

# Найти процесс-виновник
iotop            # I/O по процессам в реальном времени
pidstat -d 1     # I/O по PID (kB_rd/s, kB_wr/s)

# Найти конкретные файлы
lsof -p PID      # какие файлы открыты
```

### Причины и рецепты

| Причина | Рецепт |
|---------|--------|
| **Медленный диск (HDD)** | Заменить на SSD/NVMe |
| **Много случайных чтений** | Увеличить RAM (больше page cache), предзагрузка |
| **БД делает полный скан** | Добавить индексы |
| **Нет кэша в приложении** | Добавить Redis/Memcached |
| **Swap (нехватка RAM)** | Добавить RAM, уменьшить `vm.swappiness` |
| **Много мелких записей** | Буферизация, асинхронная запись, write batching |
| **Журналирование (WAL)** | SSD для WAL, настройка `sync_commit` |
| **Сеть как I/O** | CDN, кэш ответов, сжатие трафика |

### Рецепт 1: Проверить наличие индексов (для БД)

```sql
-- Медленно: полный скан таблицы
EXPLAIN SELECT * FROM orders WHERE customer_id = 12345;
-- Seq Scan on orders (cost=0.00..45000.00) -- полный скан!

-- Добавить индекс
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Теперь быстро
EXPLAIN SELECT * FROM orders WHERE customer_id = 12345;
-- Index Scan using idx_orders_customer (cost=0.43..8.45) -- индекс!
```

### Рецепт 2: Кэширование на уровне приложения

```java
// Redis-кэш для результатов запросов к БД
String cacheKey = "product:" + productId;
Product product = redis.get(cacheKey, Product.class);
if (product == null) {
    product = db.findProductById(productId);  // дорогой I/O
    redis.set(cacheKey, product, 300);  // кэшировать 5 минут
}
return product;
```

### Рецепт 3: Уменьшить vm.swappiness

```bash
# По умолчанию swappiness=60 → ядро агрессивно свопит
sysctl vm.swappiness=10   # меньше свопинга, RAM отдаётся приложениям
echo "vm.swappiness=10" >> /etc/sysctl.conf
```

### Рецепт 4: I/O Scheduler

```bash
cat /sys/block/sda/queue/scheduler
# [mq-deadline] kyber bfq none

# Для SSD — none (noop): нет смысла оптимизировать порядок запросов
echo none > /sys/block/nvme0n1/queue/scheduler

# Для HDD с разными приоритетами — bfq: честное планирование
echo bfq > /sys/block/sda/queue/scheduler
```

---

## Подробно

### Как отличить «диск медленный» от «диска мало»

**Диск перегружен (`%util = 100%`):** I/O запросы стоят в очереди (avgqu-sz > 1). Решение: более быстрый диск или RAID.

**Диска достаточно, но данных нет в кэше:** page cache маленький, каждое чтение идёт на диск. Решение: добавить RAM.

```bash
iostat -xz 1
Device   r/s  w/s  await  %util  avgqu-sz
sda      150   50   45.0   95      3.2
              ↑                    ↑
          много чтений         очередь растёт → диск перегружен
```

### Read-ahead (предзагрузка)

При последовательном чтении ядро автоматически загружает следующие блоки до запроса. Для random access — вредно (загружает ненужное):

```bash
# Настроить read-ahead в 512 КБ (для последовательных паттернов)
blockdev --setra 1024 /dev/sda  # 1024 * 512 байт = 512 КБ

# Для БД с random I/O — уменьшить или отключить:
blockdev --setra 0 /dev/nvme0n1
```

### Асинхронный I/O (AIO, io_uring)

Вместо блокирующего `read()` — неблокирующий вариант: отправить запрос на I/O и продолжить работу; получить результат по завершении.

**io_uring** (Linux 5.1+) — современный механизм асинхронного I/O с минимальным числом syscall:
```
Заполнить кольцо запросов (без syscall)
→ Один io_uring_enter() для отправки
→ Продолжить работу
→ Проверить кольцо результатов (без syscall)
```

### Columnar storage для аналитики

Аналитические запросы читают только нужные столбцы. Row-store (PostgreSQL) читает всю строку. Column-store (Parquet, ClickHouse) читает только столбцы из SELECT:

```sql
-- Из таблицы с 100 столбцами читаем только 2:
SELECT AVG(price), COUNT(*) FROM orders WHERE year = 2024;
-- Row-store: читает все 100 столбцов каждой строки
-- Column-store: читает только столбцы price, year → в 50× меньше I/O
```
