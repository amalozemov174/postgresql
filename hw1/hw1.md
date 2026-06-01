# Домашнее задание 1 (HW1)

## Окружение
* **ОС:** Virtual Machine (VM) RHEL 8
* **RAM:** 7.5 GiB
* **CPU:** 4 cores

---

## 1. Установка PostgreSQL 17
Выполнялась на рабочем месте. Доступы до официального репозитория с PostgreSQL 17 закрыты, поэтому пакеты устанавливались вручную из локальных RPM-файлов:

```bash
dnf localinstall postgresql17-17.5-3PGDG.rhel8.x86_64.rpm
dnf localinstall postgresql17-contrib-17.5-3PGDG.rhel8.x86_64.rpm
dnf localinstall postgresql17-devel-17.5-3PGDG.rhel8.x86_64.rpm
dnf localinstall postgresql17-docs-17.5-3PGDG.rhel8.x86_64.rpm
dnf localinstall postgresql17-libs-17.5-3PGDG.rhel8.x86_64.rpm
dnf localinstall postgresql17-pltcl-17.5-3PGDG.rhel8.x86_64.rpm
dnf localinstall postgresql17-server-17.5-3PGDG.rhel8.x86_64.rpm
```

---

## 2. Аудит настроек операционной системы

### Проверка swappiness
Используется стандартное значение `30`.
```bash
[root@tdb-txwb2 ~]# cat /proc/sys/vm/swappiness
30
```

### Проверка версии ОС
```bash
[root@tdb-txwb2 ~]# sudo cat /proc/version
Linux version 4.18.0-513.9.1.el8_9.x86_64 (mockbuild@x86-vm-09.build.eng.bos.redhat.com) (gcc version 8.5.0 20210514 (Red Hat 8.5.0-20) (GCC)) #1 SMP Thu Nov 16 10:29:04 EST 2023
```

### Проверка настроек HugePages
Механизм HugePages в данный момент не используется.
```bash
[root@tdb-txwb2 ~]# grep Huge /proc/meminfo
AnonHugePages:     34816 kB
ShmemHugePages:        0 kB
FileHugePages:         0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
```

---

## 3. Дефолтные настройки PostgreSQL
```ini
max_worker_processes = 8
max_connections = 100
shared_buffers = 128MB
max_wal_size = 1GB
min_wal_size = 80MB
fsync = on
synchronous_commit = on
random_page_cost = 4
huge_pages = try
```

---

## 4. Тестирование производительности (pgBench) до тюнинга

### Тест №1: Простой запуск (1 клиент, 1 поток)
```bash
[postgres@tdb-txwb2 ~]\$ pgbench -P 1 -T 10 postgres
pgbench (17.5)
starting vacuum...end.
progress: 1.0 s, 738.9 tps, lat 1.346 ms stddev 2.051, 0 failed
progress: 2.0 s, 788.0 tps, lat 1.269 ms stddev 0.335, 0 failed
progress: 3.0 s, 740.0 tps, lat 1.350 ms stddev 0.725, 0 failed
progress: 4.0 s, 751.0 tps, lat 1.331 ms stddev 0.169, 0 failed
progress: 5.0 s, 800.0 tps, lat 1.251 ms stddev 0.127, 0 failed
progress: 6.0 s, 798.9 tps, lat 1.250 ms stddev 0.160, 0 failed
progress: 7.0 s, 784.0 tps, lat 1.275 ms stddev 0.357, 0 failed
progress: 8.0 s, 756.1 tps, lat 1.322 ms stddev 0.287, 0 failed
progress: 9.0 s, 792.0 tps, lat 1.263 ms stddev 0.132, 0 failed
progress: 10.0 s, 782.0 tps, lat 1.277 ms stddev 0.154, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 7732
number of failed transactions: 0 (0.000%)
latency average = 1.292 ms
latency stddev = 0.705 ms
initial connection time = 4.044 ms
tps = 773.474351 (without initial connection time)
```

### Тест №2: Расширенный запуск (10 клиентов, 1 поток)
```bash
[postgres@tdb-txwb2 ~]\$ pgbench -P 1 -c 10 -T 10 postgres
pgbench (17.5)
starting vacuum...end.
progress: 1.0 s, 1494.9 tps, lat 6.420 ms stddev 3.986, 0 failed
progress: 2.0 s, 1564.0 tps, lat 6.388 ms stddev 4.083, 0 failed
progress: 3.0 s, 1362.0 tps, lat 7.335 ms stddev 4.799, 0 failed
progress: 4.0 s, 1206.8 tps, lat 8.244 ms stddev 8.265, 0 failed
progress: 5.0 s, 1094.1 tps, lat 9.152 ms stddev 8.845, 0 failed
progress: 6.0 s, 1056.1 tps, lat 9.402 ms stddev 9.433, 0 failed
progress: 7.0 s, 1229.7 tps, lat 8.146 ms stddev 6.776, 0 failed
progress: 8.0 s, 1309.3 tps, lat 7.648 ms stddev 6.289, 0 failed
progress: 9.0 s, 1221.0 tps, lat 8.152 ms stddev 6.498, 0 failed
progress: 10.0 s, 1278.2 tps, lat 7.767 ms stddev 6.536, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 12829
number of failed transactions: 0 (0.000%)
latency average = 7.765 ms
latency stddev = 6.679 ms
initial connection time = 33.640 ms
tps = 1284.849245 (without initial connection time)
```

### Тест №3: Расширенный запуск (10 клиентов, 4 потока)
```bash
[postgres@tdb-txwb2 ~]\$ pgbench -P 1 -c 10 -j 4 -T 10 postgres
pgbench (17.5)
starting vacuum...end.
progress: 1.0 s, 1454.8 tps, lat 6.772 ms stddev 4.012, 0 failed
progress: 2.0 s, 1542.1 tps, lat 6.480 ms stddev 3.933, 0 failed
progress: 3.0 s, 1525.0 tps, lat 6.540 ms stddev 4.113, 0 failed
progress: 4.0 s, 1506.5 tps, lat 6.651 ms stddev 4.023, 0 failed
progress: 5.0 s, 1454.3 tps, lat 6.881 ms stddev 4.752, 0 failed
progress: 6.0 s, 1521.2 tps, lat 6.566 ms stddev 3.936, 0 failed
progress: 7.0 s, 1509.9 tps, lat 6.619 ms stddev 3.986, 0 failed
progress: 8.0 s, 1452.9 tps, lat 6.880 ms stddev 4.849, 0 failed
progress: 9.0 s, 1485.2 tps, lat 6.734 ms stddev 4.222, 0 failed
progress: 10.0 s, 1450.0 tps, lat 6.893 ms stddev 4.476, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 14911
number of failed transactions: 0 (0.000%)
latency average = 6.700 ms
latency stddev = 4.240 ms
initial connection time = 11.536 ms
tps = 1491.353381 (without initial connection time)
```

### Тест №4: Расширенный запуск (10 клиентов, 4 потока, reconnection)
Каждая сессия устанавливает новое соединение.
```bash
[postgres@tdb-txwb2 ~]\$ pgbench -P 1 -c 10 -j 4 -T 10 -C postgres
pgbench (17.5)
starting vacuum...end.
progress: 1.0 s, 218.8 tps, lat 36.718 ms stddev 19.903, 0 failed
progress: 2.0 s, 189.6 tps, lat 43.421 ms stddev 31.346, 0 failed
progress: 3.0 s, 286.7 tps, lat 30.869 ms stddev 18.569, 0 failed
progress: 4.0 s, 306.0 tps, lat 28.092 ms stddev 13.917, 0 failed
progress: 5.0 s, 285.0 tps, lat 29.653 ms stddev 14.832, 0 failed
progress: 6.0 s, 292.4 tps, lat 29.492 ms stddev 14.553, 0 failed
progress: 7.0 s, 312.8 tps, lat 27.607 ms stddev 13.962, 0 failed
progress: 8.0 s, 322.6 tps, lat 26.654 ms stddev 13.513, 0 failed
progress: 9.0 s, 348.5 tps, lat 24.618 ms stddev 14.181, 0 failed
progress: 10.0 s, 392.0 tps, lat 21.793 ms stddev 12.088, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 2964
number of failed transactions: 0 (0.000%)
latency average = 28.783 ms
latency stddev = 17.305 ms
average connection time = 4.956 ms
tps = 295.962006 (including reconnection times)
```

---

## 5. Оптимизация параметров PostgreSQL (Cybertec)
Конфигурация была изменена в соответствии с рекомендациями тюнера Cybertec:

```ini
# Connectivity
max_connections = 500
superuser_reserved_connections = 3

# Memory Settings
shared_buffers = '2048 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '6 GB'
effective_io_concurrency = 200 
random_page_cost = 1.25 

# Monitoring
shared_preload_libraries = 'pg_stat_statements' 
track_io_timing = on 
track_functions = pl 

# Replication
wal_level = replica 
max_wal_senders = 0
synchronous_commit = on

# Checkpointing
checkpoint_timeout = '15 min'
checkpoint_completion_target = 0.9
max_wal_size = '1024 MB'
min_wal_size = '512 MB'

# WAL writing
wal_compression = on
wal_buffers = -1 
wal_writer_delay = 200ms
wal_writer_flush_after = 1MB

# Background writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 0

# Parallel queries
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_maintenance_workers = 2
max_parallel_workers = 4
parallel_leader_participation = on

# Advanced features
enable_partitionwise_join = on
enable_partitionwise_aggregate = on
jit = on
max_slot_wal_keep_size = '1000 MB'
track_wal_io_timing = on
maintenance_io_concurrency = 200
wal_recycle = off
```

---

## 6. Тестирование производительности после изменений

### Тест №1: 10 клиентов, 1 поток
### 1) Запуск pgbench после изменения параметров в 1 поток 10 клиентов
```bash
[postgres@tdb-txwb2 ~]\$ pgbench -P 1 -c 10 -T 10 postgres
pgbench (17.5)
starting vacuum...end.
progress: 1.0 s, 1241.9 tps, lat 7.718 ms stddev 5.345, 0 failed
progress: 2.0 s, 1395.0 tps, lat 7.171 ms stddev 4.783, 0 failed
progress: 3.0 s, 1359.0 tps, lat 7.340 ms stddev 4.781, 0 failed
progress: 4.0 s, 1362.0 tps, lat 7.332 ms stddev 4.685, 0 failed
progress: 5.0 s, 1355.1 tps, lat 7.369 ms stddev 5.059, 0 failed
progress: 6.0 s, 1427.9 tps, lat 6.994 ms stddev 4.265, 0 failed
progress: 7.0 s, 1353.0 tps, lat 7.383 ms stddev 4.756, 0 failed
progress: 8.0 s, 1336.0 tps, lat 7.472 ms stddev 4.810, 0 failed
progress: 9.0 s, 1389.0 tps, lat 7.174 ms stddev 4.804, 0 failed
progress: 10.0 s, 1291.9 tps, lat 7.739 ms stddev 4.822, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 13521
number of failed transactions: 0 (0.000%)
latency average = 7.365 ms
latency stddev = 4.818 ms
initial connection time = 33.858 ms
tps = 1354.813149 (without initial connection time)
```

### 2) Запуск pgbench после изменения параметров в 4 потока 10 клиентов с установкой нового соединения
```bash
[postgres@tdb-txwb2 ~]\$ pgbench -P 1 -c 10 -j 4 -T 10 -C postgres
pgbench (17.5)
starting vacuum...end.
progress: 1.0 s, 337.9 tps, lat 24.540 ms stddev 12.511, 0 failed
progress: 2.0 s, 335.7 tps, lat 25.178 ms stddev 12.027, 0 failed
progress: 3.0 s, 346.1 tps, lat 24.448 ms stddev 14.249, 0 failed
progress: 4.0 s, 341.3 tps, lat 25.155 ms stddev 13.288, 0 failed
progress: 5.0 s, 328.4 tps, lat 26.009 ms stddev 14.422, 0 failed
progress: 6.0 s, 342.3 tps, lat 25.054 ms stddev 12.348, 0 failed
progress: 7.0 s, 344.4 tps, lat 24.892 ms stddev 13.264, 0 failed
progress: 8.0 s, 339.0 tps, lat 24.985 ms stddev 12.479, 0 failed
progress: 9.0 s, 338.0 tps, lat 25.128 ms stddev 14.200, 0 failed
progress: 10.0 s, 348.5 tps, lat 24.506 ms stddev 12.853, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 3410
number of failed transactions: 0 (0.000%)
latency average = 24.979 ms
latency stddev = 13.196 ms
average connection time = 4.351 ms
tps = 340.551392 (including reconnection times)
```

### Отключение ACID для максимальной производительности
```ini
synchronous_commit = off
fsync = off
full_page_writes = off
```

```bash
[postgres@tdb-txwb2 ~]\$ pgbench -P 1 -c 10 -j 4 -T 10 postgres
pgbench (17.5)
starting vacuum...end.
progress: 1.0 s, 2675.9 tps, lat 3.686 ms stddev 2.133, 0 failed
progress: 2.0 s, 2813.9 tps, lat 3.552 ms stddev 2.314, 0 failed
progress: 3.0 s, 2760.8 tps, lat 3.619 ms stddev 2.246, 0 failed
progress: 4.0 s, 2813.0 tps, lat 3.556 ms stddev 2.272, 0 failed
progress: 5.0 s, 2827.4 tps, lat 3.528 ms stddev 2.200, 0 failed
progress: 6.0 s, 2832.8 tps, lat 3.527 ms stddev 2.269, 0 failed
progress: 7.0 s, 2832.9 tps, lat 3.530 ms stddev 2.225, 0 failed
progress: 8.0 s, 2712.1 tps, lat 3.689 ms stddev 2.215, 0 failed
progress: 9.0 s, 2754.8 tps, lat 3.629 ms stddev 2.377, 0 failed
progress: 10.0 s, 2863.1 tps, lat 3.490 ms stddev 2.282, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 27896
number of failed transactions: 0 (0.000%)
latency average = 3.580 ms
latency stddev = 2.257 ms
initial connection time = 11.148 ms
tps = 2789.995900 (without initial connection time)
```

### Выводы:
* На синтетических тестах не удалось добиться разницы в TPS со стандартными и "потюненными" параметрами.
* Отключение `synchronous_commit = off`, `fsync = off`, `full_page_writes = off` серьезно ускоряет работу БД. Думаю, что отключение в проде возможно для систем, где потеря данных не является критичной.

