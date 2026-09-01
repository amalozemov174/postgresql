# Домашнее задание 4 (hw4)

## Окружение

- 2 VM
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

Рекомендация от Tantor для теста на двух ВМ:

```conf
wal_compression = lz4
jit = on
client_connection_check_interval = 5s
default_toast_compression = pglz
enable_async_append = on
autovacuum_vacuum_insert_threshold = 1077
autovacuum_vacuum_insert_scale_factor = 0.01
logical_decoding_work_mem = 64MB
maintenance_io_concurrency = 4
wal_keep_size = 1086MB
hash_mem_multiplier = 2.0
max_parallel_maintenance_workers = 4
max_parallel_workers = 4
max_logical_replication_workers = 4
max_sync_workers_per_subscription = 2
autovacuum = on
autovacuum_max_workers = 4
autovacuum_work_mem = 202MB
autovacuum_naptime = 15s
autovacuum_vacuum_threshold = 1077
autovacuum_analyze_threshold = 538
autovacuum_vacuum_scale_factor = 0.001
autovacuum_analyze_scale_factor = 0.0007
autovacuum_vacuum_cost_limit = 2024
vacuum_cost_limit = 8000
autovacuum_vacuum_cost_delay = 10ms
vacuum_cost_delay = 10ms
autovacuum_freeze_max_age = 500000000
autovacuum_multixact_freeze_max_age = 800000000
shared_buffers = 1623MB
max_connections = 50
max_files_per_process = 1000
superuser_reserved_connections = 4
work_mem = 58MB
temp_buffers = 6649kB
maintenance_work_mem = 202MB
huge_pages = try
fsync = on
wal_level = replica
synchronous_commit = off
full_page_writes = on
wal_buffers = 16MB
wal_writer_delay = 203ms
wal_writer_flush_after = 1286kB
min_wal_size = 576MB
max_wal_size = 1286MB
max_replication_slots = 10
max_wal_senders = 10
wal_sender_timeout = 300s
wal_log_hints = on
hot_standby = on
wal_receiver_timeout = 300s
max_standby_streaming_delay = 1800s
hot_standby_feedback = on
wal_receiver_status_interval = 10s
checkpoint_timeout = 30min
checkpoint_warning = 30s
checkpoint_completion_target = 0.85
commit_delay = 103
commit_siblings = 10
bgwriter_delay = 50ms
bgwriter_lru_maxpages = 502
bgwriter_lru_multiplier = 7.0
effective_cache_size = 4614MB
cpu_operator_cost = 0.0025
default_statistics_target = '500'
random_page_cost = 2.5
seq_page_cost = 1
join_collapse_limit = 9
from_collapse_limit = 9
geqo = on
geqo_threshold = 12
effective_io_concurrency = 4
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_locks_per_transaction = 80
max_pred_locks_per_transaction = 80
statement_timeout = 86400000
idle_in_transaction_session_timeout = 86400000
```

### 6. Разворачивание БД «тайские авиалинии» среднего размера на первой db1(cluster_name='db1')
```bash
[postgres@tdb-txwb2 ~]$ pg_restore -U postgres -d thai thai.sql
```
### 7. Установка pmm server
```bash
curl -fsSL https://raw.githubusercontent.com/percona/pmm/refs/heads/main/get-pmm.sh | /bin/bash
```
### 8. Установка pmm клиента

```bash
[root@tdb-etcd1 ~]# dnf localinstall pmm-client-3.9.0-1.el8.x86_64.rpm
```
### 9. Установка pg_stat_momnitor
```bash
[root@tdb-etcd1 ~]# dnf localinstall pg_stat_monitor_18-2.3.2-1PGDG.rhel8.10.x86_64.rpm
```
### 10. Подкдючаемся к серверу pmm
```bash
[root@tdb-etcd1 ~]# pmm-admin config   --force   --server-insecure-tls   --server-url='https://admin:Compass12@10.222.1.62:443'
```


#### Проверяем статус
```bash
проверяем статус
[root@tdb-etcd1 ~]# pmm-admin status
Agent ID : b7b8a430-5c5e-4275-a898-2386aed618d9
Node ID  : 8ed52a14-6dc7-4482-b7a7-571a2ecab50a
Node name: tdb-etcd1

PMM Server:
        URL    : https://10.222.1.62:443/
        Version: 3.9.0

PMM Client:
        Connected        : true
        Time drift       : -11.000629604s
        Latency          : 471.534µs
        Connection uptime: 100
        pmm-admin version: 3.9.0
        pmm-agent version: 3.9.0
Agents:
        156a1f3f-4919-4b84-9c73-20bbe1758d5a vmagent                        Running        42000
        481a5c61-ef66-4faa-8236-4e87608ba003 node_exporter                  Running        42001
```

#### создаем расширение
```bash
psql -d postgres -c "CREATE EXTENSION pg_stat_monitor;"
```
### 10. Скриншоты работы с pmm

#### Регистрация сервиса

![Регистрация сервиса в web админке pmm](register.png)

#### Проверка статуса сервиса pmm-client

![статус](service_added.png)

#### Нагрузка select

![статус](select_load.png)

#### Нагрузка insert

![статус](transaction_screen.png)


Наблюдения:
 - pmm легко устанавливается
 - из коробки готовые дашборды, например, в отличии от zabbix
 - За монитрирнг платим ресурсами, в среднем нагрузка на ядро увеличилась на 0,2, на средненагруженной системе
 - хотелось бы дашбор с информацией по сессиям(активные/блокирующие) + больше информации по запросам
 
 