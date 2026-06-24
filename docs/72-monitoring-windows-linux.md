# Билет 72. Мониторинг производительности: Windows и Linux

## Ответ

### Linux — основные инструменты

| Инструмент | Что мониторит | Пример |
|-----------|---------------|--------|
| `top` / `htop` | CPU, RAM по процессам | `top -d 1` |
| `vmstat` | CPU, память, swap, I/O | `vmstat 1 10` |
| `iostat` | Диски (IOPS, пропускная способность) | `iostat -xz 1` |
| `sar` | История метрик (CPU, память, сеть) | `sar -u 1 10` |
| `netstat` / `ss` | Сетевые соединения и статистика | `ss -tuln` |
| `free` | RAM и swap | `free -m` |
| `pidstat` | CPU/I/O конкретного процесса | `pidstat -u 1` |
| `dmesg` | Сообщения ядра (OOM, ошибки) | `dmesg -T | tail -20` |

### Windows — основные инструменты

| Инструмент | Что мониторит |
|-----------|---------------|
| **Task Manager** (Ctrl+Shift+Esc) | Быстрый обзор: CPU, RAM, диск, сеть по процессам |
| **Resource Monitor** (resmon) | Детали по CPU, памяти, диску, сети |
| **Performance Monitor** (perfmon) | Настраиваемые счётчики производительности; запись в файл |
| **Process Explorer** (Sysinternals) | Расширенный Task Manager: дерево процессов, хэндлы, DLL |
| **Process Monitor** (Sysinternals) | Трассировка файловых, реестровых, сетевых операций |
| **WinDbg** | Отладка и анализ дампов памяти |
| **perfmon /report** | Автоматический отчёт о производительности |

### Ключевые счётчики Windows (perfmon)

```
Processor(_Total)\% Processor Time   — общая загрузка CPU
Memory\Available MBytes              — доступная RAM
Memory\Pages/sec                     — подкачка (> 20 = проблема)
PhysicalDisk(_Total)\% Disk Time     — загрузка диска
PhysicalDisk(_Total)\Avg. Disk Queue Length  — очередь (> 2 = проблема)
Network Interface\Bytes Total/sec    — сетевой трафик
```

### Сравнение подходов

| Аспект | Linux | Windows |
|--------|-------|---------|
| CLI-инструменты | Богатые (top, vmstat, iostat, sar) | Ограниченные (typeperf, wmic) |
| GUI | htop, Grafana | Task Manager, perfmon, Resource Monitor |
| История метрик | sar, Prometheus | perfmon Data Collector Sets |
| Трассировка | strace, perf, eBPF | Process Monitor, ETW |

---

## Подробно

### sar — системный аудит

`sar` (System Activity Reporter) записывает метрики регулярно (по умолчанию каждые 10 минут) и позволяет анализировать историю:

```bash
sar -u           # история CPU сегодня
sar -u -f /var/log/sa/sa15  # CPU за 15-е число
sar -r           # история памяти
sar -d           # история дисков
sar -n DEV       # история сети
```

На production-серверах `sar` — первый инструмент для анализа «что происходило вчера в 3 ночи».

### Windows: Event Viewer vs Performance Monitor

**Event Viewer** (`eventvwr`) — журнал событий: ошибки ОС, падения приложений, проблемы безопасности. При крашах ищем здесь.

**Performance Monitor** — числовые метрики во времени. Можно создать **Data Collector Set** для записи счётчиков в файл и последующего анализа.

### Windows: typeperf — CLI мониторинг

```powershell
typeperf "\Processor(_Total)\% Processor Time" -si 1 -sc 10
# показывает загрузку CPU каждую секунду, 10 раз
```

### eBPF (Linux) — современный инструмент трассировки

eBPF (Extended Berkeley Packet Filter) — механизм ядра для безопасного запуска кода в ядре без модулей:

```bash
bpftool            # работа с eBPF программами
bcc-tools/funccount # считать вызовы функций
bcc-tools/execsnoop # новые процессы в реальном времени
bcc-tools/opensnoop # открытые файлы
```

Инструменты bcc позволяют трассировать ядро с минимальным overhead, без strace (который замедляет в 10–100×).

### Sysinternals Suite (Windows)

Набор от Марка Руссиновича (Microsoft). Самые полезные:
- **Process Explorer**: видит DLL, хэндлы, TCP-соединения процесса.
- **Process Monitor**: записывает всё: каждый read/write файла, каждый доступ к реестру.
- **Autoruns**: что запускается при старте системы.
- **TCPView**: все активные соединения + процесс.
