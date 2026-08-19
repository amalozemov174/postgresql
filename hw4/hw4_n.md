# Домашнее задание 4 (hw4)

## Окружение

- 3 VM
- ОС: RHEL 8
- RAM: 7.5 GiB
- CPU: 4 core

## Конфигурация PostgreSQL и окруженияф

### 1. Установка PostgreSQL 18.4 на тесте для трех ВМ

```bash
dnf install postgresql18
dnf install postgresql18-contrib
dnf install postgresql18-devel
dnf install postgresql18-docs
dnf install postgresql18-libs
dnf install postgresql18-pltcl
dnf install postgresql18-server
```

### 2. Проверка swappiness на тесте для двух ВМ

```bash
[root@tdb-txwb2 ~]# cat /proc/sys/vm/swappiness
30
```

### 3. Проверка версии ОС на тесте для двух ВМ

```bash
[root@tdb-txwb2 ~]# sudo cat /proc/version
Linux version 4.18.0-513.9.1.el8_9.x86_64 (mockbuild@x86-vm-09.build.eng.bos.redhat.com) (gcc version 8.5.0 20210514 (Red Hat 8.5.0-20) (GCC)) #1 SMP Thu Nov 16 10:29:04 EST 2023
```

### 4. Проверка HugePages на тесте для двух ВМ

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

### 5. Настройки БД

Рекомендация Cybertec для теста на двух ВМ:

```conf
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

Дополнительные настройки:

```conf
shared_buffers = '8192 MB'
work_mem = '32 MB'
maintenance_work_mem = '420 MB'
effective_cache_size = '22 GB'
effective_io_concurrency = 100
random_page_cost = 1.25
```

```conf
# Connectivity
max_connections = 1000
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

### 6. Разворачивание БД «тайские авиалинии» среднего размера на первой db1(cluster_name='db1')
```bash
[postgres@tdb-txwb2 ~]$ pg_restore -U postgres -d thai thai.sql
```
### 7. Создаем стендбай слот slot1
```sql
SELECT pg_create_physical_replication_slot('slot1');
```
### 8. Создаем скрипты для нагрузки бд

Запросы
```bash
[postgres@tdb-txwb2 ~]$ cat workload_select.sql
\set r random(1, 5000000)
SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;
```
Вставки
```sql
INSERT INTO book.tickets (fkRide, fio, contact, fkSeat)
VALUES (
    ceil(random()*100),
    (array(SELECT fam FROM book.fam))[ceil(random()*110)]::text || ' ' ||
    (array(SELECT nam FROM book.nam))[ceil(random()*110)]::text,
    ('{"phone":"+7' || (1000000000::bigint + floor(random()*9000000000)::bigint)::text || '"}')::jsonb,
    ceil(random()*100)
);
```
### 9. Нагрузка бд без стендбаев

#### Запросы
```bash
[postgres@tdb-etcd1 ~]$ for i in 100 200 300 400 500 600 700; do echo "connections:" $i; pgbench -c $i -j 4 -T 10 -f ~/workload_select.sql -U postgres -h 10.222.1.131 -p 5432 thai -n | grep -E "latency|tps"; done
connections: 100
latency average = 11.364 ms
tps = 8799.386170 (without initial connection time)
connections: 200
latency average = 24.034 ms
tps = 8321.513446 (without initial connection time)
connections: 300
latency average = 37.928 ms
tps = 7909.783970 (without initial connection time)
connections: 400
latency average = 52.103 ms
tps = 7677.058371 (without initial connection time)
connections: 500
latency average = 67.991 ms
tps = 7353.920140 (without initial connection time)
connections: 600
latency average = 80.964 ms
tps = 7410.695302 (without initial connection time)
connections: 700
latency average = 98.933 ms
tps = 7075.524824 (without initial connection time)
```

#### Вставка
```bash
[postgres@tdb-etcd1 ~]$ for i in 100 200 300 400 500 600 700; do echo "connections:" $i; pgbench -c $i -j 4 -T 10 -f ~/workload_insert.sql -U postgres -h 10.222.1.131 -p 5432 thai -n | grep -E "latency|tps"; done
connections: 100
latency average = 27.504 ms
tps = 3635.786623 (without initial connection time)
connections: 200
latency average = 60.833 ms
tps = 3287.667325 (without initial connection time)
connections: 300
latency average = 97.936 ms
tps = 3063.237941 (without initial connection time)
connections: 400
latency average = 137.361 ms
tps = 2912.040622 (without initial connection time)
connections: 500
latency average = 184.180 ms
tps = 2714.732272 (without initial connection time)
connections: 600
latency average = 227.016 ms
tps = 2642.980438 (without initial connection time)
connections: 700
latency average = 303.051 ms
tps = 2309.842987 (without initial connection time)
```
### 10. Создаем один асинхронный стендбай и 1 синхронный стендбай

#### синхронный стендбай
```bash
pg_basebackup -D /postgresql/18 -n -P -R --slot=slot1 -h 10.222.1.131 -p 5432 -c fast
```
```bash
cluster_name='db2'
```bash

на мастере db1 выполняем настройки

```sql
postgres=# ALTER SYSTEM SET synchronous_commit='remote_apply';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET  synchronous_standby_names ='db2';
ALTER SYSTEM

postgres=# select * from pg_stat_replication\gx
-[ RECORD 1 ]----+------------------------------
pid              | 38480
usesysid         | 10
usename          | postgres
application_name | db2
client_addr      | 10.222.1.132
client_hostname  |
client_port      | 48564
backend_start    | 2026-08-06 15:28:48.536297+05
backend_xmin     |
state            | streaming
sent_lsn         | 0/30000D8
write_lsn        | 0/30000D8
flush_lsn        | 0/30000D8
replay_lsn       | 0/30000D8
write_lag        | 00:00:00.000217
flush_lag        | 00:00:00.000217
replay_lag       | 00:00:00.000217
sync_priority    | 1
sync_state       | sync
reply_time       | 2026-08-06 15:28:45.371076+05
```
#### асинхронный стендбай
```bash
pg_basebackup -D /postgresql/18 -n -P -R --slot=slot2 -h 10.222.1.131 -p 5432 -c fast
```
```bash
cluster_name='db3'
```bash

на мастере db1 выполняем настройки

```sql
test=# select * from pg_stat_replication\gx
-[ RECORD 1 ]----+------------------------------
pid              | 72143
usesysid         | 10
usename          | postgres
application_name | db2
client_addr      | 10.222.1.132
client_hostname  |
client_port      | 34300
backend_start    | 2026-08-07 13:49:42.055979+05
backend_xmin     |
state            | streaming
sent_lsn         | 0/5000060
write_lsn        | 0/5000060
flush_lsn        | 0/5000060
replay_lsn       | 0/5000060
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 1
sync_state       | sync
reply_time       | 2026-08-07 14:09:13.978602+05
-[ RECORD 2 ]----+------------------------------
pid              | 72640
usesysid         | 10
usename          | postgres
application_name | db3
client_addr      | 10.222.1.133
client_hostname  |
client_port      | 47086
backend_start    | 2026-08-07 14:08:53.493083+05
backend_xmin     |
state            | streaming
sent_lsn         | 0/5000060
write_lsn        | 0/5000060
flush_lsn        | 0/5000060
replay_lsn       | 0/5000060
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2026-08-07 14:09:14.499474+05
```
#### Нагрузка на бд с 2 стендбаями

##### Селекты
```bash
[postgres@tdb-etcd1 ~]$ for i in 100 200 300 400 500 600 700; do
> echo "connections:" $i
> pgbench -c $i -j 4 -T 10 -f ~/workload_select.sql -U postgres -h 10.222.1.131 -p 5432 thai -n | grep -E "latency|tps"
> done
connections: 100
latency average = 113.822 ms
tps = 878.566879 (without initial connection time)
connections: 200
latency average = 103.788 ms
tps = 1927.005592 (without initial connection time)
connections: 300
latency average = 75.493 ms
tps = 3973.890288 (without initial connection time)
connections: 400
latency average = 59.207 ms
tps = 6755.998750 (without initial connection time)
connections: 500
latency average = 71.466 ms
tps = 6996.374549 (without initial connection time)
connections: 600
latency average = 86.523 ms
tps = 6934.535894 (without initial connection time)
connections: 700
latency average = 101.263 ms
tps = 6912.708119 (without initial connection time)
```
##### Вставка
```bash
[postgres@tdb-etcd1 ~]$ for i in 100 200 300 400 500 600 700; do
> echo "connections:" $i
> pgbench -c $i -j 4 -T 10 -f ~/workload_insert.sql -U postgres -h 10.222.1.131 -p 5432 thai -n | grep -E "latency|tps"
> done
connections: 100
latency average = 31.787 ms
tps = 3145.940385 (without initial connection time)
connections: 200
latency average = 65.481 ms
tps = 3054.330526 (without initial connection time)
connections: 300
latency average = 105.402 ms
tps = 2846.252891 (without initial connection time)
connections: 400
latency average = 147.951 ms
tps = 2703.593974 (without initial connection time)
connections: 500
latency average = 195.153 ms
tps = 2562.086882 (without initial connection time)
connections: 600
latency average = 247.284 ms
tps = 2426.360430 (without initial connection time)
connections: 700
latency average = 309.496 ms
tps = 2261.741345 (without initial connection time)
```

##### Эксперимент с отключением синхронного стендбая

Отключаю синхронный стендбай
```bash
systemct stop postgresql-18
```
На мастере выполняю вставку, и вижк зависание в логе сообщение
```bash
thai=# INSERT INTO book.tickets (fkRide, fio, contact, fkSeat)
thai-# VALUES (
thai(#     ceil(random()*100),
thai(#     (array(SELECT fam FROM book.fam))[ceil(random()*110)]::text || ' ' ||
thai(#     (array(SELECT nam FROM book.nam))[ceil(random()*110)]::text,
thai(#     ('{"phone":"+7' || (1000000000::bigint + floor(random()*9000000000)::bigint)::text || '"}')::jsonb,
thai(#     ceil(random()*100)
thai(# );
^CCancel request sent
WARNING:  canceling wait for synchronous replication due to user request
DETAIL:  The transaction has already committed locally, but might not have been replicated to the standby.

2026-08-12 10:25:54.955 +05 [89575] LOG:  invalid resource manager ID 56 at 2/380D9B70
2026-08-12 10:25:56.026 +05 [178731] LOG:  started streaming WAL from primary at 2/38000000 on timeline 1
2026-08-12 10:25:56.037 +05 [89575] WARNING:  hot standby is not possible because of insufficient parameter settings
2026-08-12 10:25:56.037 +05 [89575] DETAIL:  max_connections = 50 is a lower setting than on the primary server, where its value was 1000.
2026-08-12 10:25:56.037 +05 [89575] CONTEXT:  WAL redo at 2/380D9B70 for XLOG/PARAMETER_CHANGE: max_connections=1000 max_worker_processes=4 max_wal_senders=10 max_prepared_xacts=0 max_locks_per_xact=98 wal_level=replica wal_log_hints=on track_commit_timestamp=off
2026-08-12 10:25:56.038 +05 [89575] LOG:  recovery has paused
```
### 11. Создаем один асинхронный стендбай и 1 асинхронный с каскадной репликацией со стендбая

Выполняю на db3 следующие настройки т.е. отключаюсь от db1 на db2:
synchronous_commit = 'remote_apply'
synchronous_standby_names = 'db2'
primary_conninfo = 'user=postgres passfile=''/var/lib/pgsql/.pgpass'' channel_binding=prefer host=10.222.1.132 port=5432 sslmode=prefer sslnegotiation=postgres sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable'
primary_slot_name = 'slot2'

Проверяю что стендбай подключился:
```sql
thai=# select * from pg_replication_slots\gx
-[ RECORD 1 ]-------+-----------
slot_name           | slot2
plugin              |
slot_type           | physical
datoid              |
database            |
temporary           | f
active              | t
active_pid          | 181767
xmin                | 177817
catalog_xmin        |
restart_lsn         | 2/3CFF8BC0
confirmed_flush_lsn |
wal_status          | reserved
safe_wal_size       |
two_phase           | f
two_phase_at        |
inactive_since      |
conflicting         |
invalidation_reason |
failover            | f
synced              | f
```

#### Проверяю нагрузку
```bash
bash-4.4$ for i in 100 200 300 400 500 600 700; do echo "connections:" $i; pgbench -c $i -j 4 -T 10 -f ~/workload_select.sql -U postgres -h 10.222.1.131 -p 5432 thai -n | grep -E "latency|tps"; done
connections: 100
latency average = 78.765 ms
tps = 1269.591830 (without initial connection time)
connections: 200
latency average = 64.785 ms
tps = 3087.124707 (without initial connection time)
connections: 300
latency average = 61.154 ms
tps = 4905.653219 (without initial connection time)
connections: 400
latency average = 55.009 ms
tps = 7271.483301 (without initial connection time)
connections: 500
latency average = 68.998 ms
tps = 7246.550165 (without initial connection time)
connections: 600
latency average = 84.919 ms
tps = 7065.569331 (without initial connection time)
connections: 700
latency average = 105.952 ms
tps = 6606.772764 (without initial connection time)
```
```bash
bash-4.4$ for i in 100 200 300 400 500 600 700; do echo "connections:" $i; pgbench -c $i -j 4 -T 10 -f ~/workload_insert.sql -U postgres -h 10.222.1.131 -p 5432 thai -n | grep -E "latency|tps"; done
connections: 100
latency average = 45.944 ms
tps = 2176.569646 (without initial connection time)
connections: 200
latency average = 93.008 ms
tps = 2150.361673 (without initial connection time)
connections: 300
latency average = 102.598 ms
tps = 2924.041611 (without initial connection time)
connections: 400
latency average = 146.997 ms
tps = 2721.137402 (without initial connection time)
connections: 500
latency average = 192.084 ms
tps = 2603.022583 (without initial connection time)
connections: 600
latency average = 239.892 ms
tps = 2501.121733 (without initial connection time)
connections: 700
latency average = 307.497 ms
tps = 2276.443970 (without initial connection time)
```
Наблюдения:
 - Для postgresql стендбаи создаются не сложно с точки зрения синтаксиса, например, по сравенению с Oracle
 - Любой стендбай уменьшает производительность бд
 - Сихронный стендбай самый безопасный с точки зрения сохраения данных, но и самый требовательный к ресурасм(ОС/настроки бд/приложение/администратор)
 - Хотя бы 1 стендбай должен быть
 
 