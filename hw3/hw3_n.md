# Домашнее задание 3 (hw3)

## Окружение

- 2 VM
- ОС: RHEL 8
- RAM: 7.5 GiB
- CPU: 4 core

## Конфигурация PostgreSQL и окруженияф

### 1. Установка PostgreSQL 17.5 на тесте для двух ВМ

На тестовой системе в закрытом DMZ-контуре, так как доступы до репозитория PostgreSQL 17 закрыты, пакеты установлены вручную:

```bash
dnf localinstall postgresql17-17.5-3PGDG.rhel8.x86_64.rpm
dnf localinstall postgresql17-contrib-17.5-3PGDG.rhel8.x86_64.rpm
dnf localinstall postgresql17-devel-17.5-3PGDG.rhel8.x86_64.rpm
dnf localinstall postgresql17-docs-17.5-3PGDG.rhel8.x86_64.rpm
dnf localinstall postgresql17-libs-17.5-3PGDG.rhel8.x86_64.rpm
dnf localinstall postgresql17-pltcl-17.5-3PGDG.rhel8.x86_64.rpm
dnf localinstall postgresql17-server-17.5-3PGDG.rhel8.x86_64.rpm
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

### 6. Разворачивание БД «тайские авиалинии» среднего размера

```bash
[postgres@tdb-txwb2 ~]$ pg_restore -U postgres -d thai thai.sql
```

## Проверка производительности по заданию

### Выбираем таблицу для партиционирования

```sql
select count(*) from book.$table_name$
```
подходят две таблицы
```sql
select count(*) from book.ride
1 500 000
```
```sql
select count(*) from book.tickets
53 997 475
```

Запрос без партиций
```sql
explain analyze select tickets.id, ride.id, bus.model from book.tickets 
inner join book.ride on tickets.id = ride.id
inner join book.bus on ride.fkbus = bus.id
where ride.startdate between '2001-05-26'::date and '2002-05-26'::date


"Hash Join  (cost=5.83..87029.88 rows=551113 width=44) (actual time=259.813..744.485 rows=549000 loops=1)"
"  Hash Cond: (ride.fkbus = bus.id)"
"  ->  Merge Join  (cost=4.72..84355.86 rows=551113 width=16) (actual time=259.776..644.121 rows=549000 loops=1)"
"        Merge Cond: (tickets.id = ride.id)"
"        ->  Index Only Scan using tickets_pkey on tickets  (cost=0.56..1057756.08 rows=56213318 width=8) (actual time=0.019..162.829 rows=1315501 loops=1)"
"              Heap Fetches: 0"
"        ->  Index Scan using ride_pkey on ride  (cost=0.43..43305.68 rows=551113 width=8) (actual time=131.211..303.970 rows=549000 loops=1)"
"              Filter: ((startdate >= '2001-05-26'::date) AND (startdate <= '2002-05-26'::date))"
"              Rows Removed by Filter: 951000"
"  ->  Hash  (cost=1.05..1.05 rows=5 width=36) (actual time=0.026..0.028 rows=5 loops=1)"
"        Buckets: 1024  Batches: 1  Memory Usage: 9kB"
"        ->  Seq Scan on bus  (cost=0.00..1.05 rows=5 width=36) (actual time=0.020..0.022 rows=5 loops=1)"
"Planning Time: 0.388 ms"
"Execution Time: 761.014 ms"
```

Выберем таблицу ride для партиционирования

выполняем партиционирование ride
```sql
CREATE TABLE book.ride_p (id integer not null, startdate date, fkbus integer, fkschedule integer) PARTITION BY RANGE(startdate);
CREATE TABLE table_2020_02 PARTITION OF table FOR VALUES FROM ('2021-02-01') TO ('2021-03-01')

select min(startdate), max(startdate) from book.ride
"2000-01-01"
"2002-09-26"

ALTER TABLE book.ride_2000_05 
ADD CONSTRAINT ride_2000_05_pkey PRIMARY KEY (id);

ALTER TABLE book.ride_2000_05
ADD CONSTRAINT ride_2000_05_ride_fkschedule_fkey 
FOREIGN KEY (fkschedule) REFERENCES book.schedule(id);

ALTER TABLE book.ride_2000_05 
ADD CONSTRAINT ride_2000_05_ride_fkbus_fkey 
FOREIGN KEY (fkbus) REFERENCES book.bus(id);

CREATE TABLE book.ride_2000_01 PARTITION OF book.ride_p FOR VALUES FROM ('2000-01-01') TO ('2000-02-01');
CREATE TABLE book.ride_2000_02 PARTITION OF book.ride_p FOR VALUES FROM ('2000-02-01') TO ('2000-03-01');
CREATE TABLE book.ride_2000_03 PARTITION OF book.ride_p FOR VALUES FROM ('2000-03-01') TO ('2000-04-01');
CREATE TABLE book.ride_2000_04 PARTITION OF book.ride_p FOR VALUES FROM ('2000-04-01') TO ('2000-05-01');
CREATE TABLE book.ride_2000_05 PARTITION OF book.ride_p FOR VALUES FROM ('2000-05-01') TO ('2000-06-01');
CREATE TABLE book.ride_2000_06 PARTITION OF book.ride_p FOR VALUES FROM ('2000-06-01') TO ('2000-07-01');
CREATE TABLE book.ride_2000_07 PARTITION OF book.ride_p FOR VALUES FROM ('2000-07-01') TO ('2000-08-01');
CREATE TABLE book.ride_2000_08 PARTITION OF book.ride_p FOR VALUES FROM ('2000-08-01') TO ('2000-09-01');
CREATE TABLE book.ride_2000_09 PARTITION OF book.ride_p FOR VALUES FROM ('2000-09-01') TO ('2000-10-01');
CREATE TABLE book.ride_2000_10 PARTITION OF book.ride_p FOR VALUES FROM ('2000-10-01') TO ('2000-11-01');
CREATE TABLE book.ride_2000_11 PARTITION OF book.ride_p FOR VALUES FROM ('2000-11-01') TO ('2000-12-01');
CREATE TABLE book.ride_2000_12 PARTITION OF book.ride_p FOR VALUES FROM ('2000-12-01') TO ('2001-01-01');
```

Запрос уходит в 1 партицию
```sql
"Seq Scan on ride_2000_01 ride_p  (cost=0.00..833.25 rows=1534 width=16) (actual time=0.021..3.118 rows=1500 loops=1)"
"  Filter: (startdate = '2000-01-01'::date)"
"  Rows Removed by Filter: 45000"
"Planning Time: 0.144 ms"
"Execution Time: 3.183 ms"
```
Запрос уходит в 10 партиций
```sql
"Append  (cost=0.00..12556.33 rows=458967 width=16) (actual time=0.029..100.943 rows=459000 loops=1)"
"  ->  Seq Scan on ride_2000_01 ride_p_1  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.027..5.346 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2000-11-01'::date))"
"  ->  Seq Scan on ride_2000_02 ride_p_2  (cost=0.00..888.50 rows=43500 width=16) (actual time=0.029..5.187 rows=43500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2000-11-01'::date))"
"  ->  Seq Scan on ride_2000_03 ride_p_3  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.029..5.721 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2000-11-01'::date))"
"  ->  Seq Scan on ride_2000_04 ride_p_4  (cost=0.00..919.00 rows=45000 width=16) (actual time=0.035..5.990 rows=45000 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2000-11-01'::date))"
"  ->  Seq Scan on ride_2000_05 ride_p_5  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.058..6.651 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2000-11-01'::date))"
"  ->  Seq Scan on ride_2000_06 ride_p_6  (cost=0.00..919.00 rows=45000 width=16) (actual time=0.050..5.933 rows=45000 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2000-11-01'::date))"
"  ->  Seq Scan on ride_2000_07 ride_p_7  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.048..6.414 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2000-11-01'::date))"
"  ->  Seq Scan on ride_2000_08 ride_p_8  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.035..8.364 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2000-11-01'::date))"
"  ->  Seq Scan on ride_2000_09 ride_p_9  (cost=0.00..919.00 rows=45000 width=16) (actual time=0.049..5.816 rows=45000 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2000-11-01'::date))"
"  ->  Seq Scan on ride_2000_10 ride_p_10  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.037..7.005 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2000-11-01'::date))"
"  ->  Seq Scan on ride_2000_11 ride_p_11  (cost=0.00..919.00 rows=1467 width=16) (actual time=0.036..4.541 rows=1500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2000-11-01'::date))"
"        Rows Removed by Filter: 43500"
"Planning Time: 5.140 ms"
"Execution Time: 115.993 ms"
```

Перебор всех партиций
```sql
"Append  (cost=0.00..38244.39 rows=1500027 width=16) (actual time=0.023..326.739 rows=1500000 loops=1)"
"  ->  Seq Scan on ride_2000_01 ride_p_1  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.022..5.334 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2000_02 ride_p_2  (cost=0.00..888.50 rows=43500 width=16) (actual time=0.050..5.334 rows=43500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2000_03 ride_p_3  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.038..5.837 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2000_04 ride_p_4  (cost=0.00..919.00 rows=45000 width=16) (actual time=0.044..5.655 rows=45000 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2000_05 ride_p_5  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.039..5.779 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2000_06 ride_p_6  (cost=0.00..919.00 rows=45000 width=16) (actual time=0.026..6.947 rows=45000 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2000_07 ride_p_7  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.040..8.580 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2000_08 ride_p_8  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.048..8.641 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2000_09 ride_p_9  (cost=0.00..919.00 rows=45000 width=16) (actual time=0.055..7.977 rows=45000 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2000_10 ride_p_10  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.027..8.091 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2000_11 ride_p_11  (cost=0.00..919.00 rows=45000 width=16) (actual time=0.059..8.079 rows=45000 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2000_12 ride_p_12  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.042..8.674 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2001_01 ride_p_13  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.059..6.622 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2001_02 ride_p_14  (cost=0.00..858.00 rows=42000 width=16) (actual time=0.064..4.912 rows=42000 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2001_03 ride_p_15  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.048..5.533 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2001_04 ride_p_16  (cost=0.00..919.00 rows=45000 width=16) (actual time=0.029..5.424 rows=45000 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2001_05 ride_p_17  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.047..5.677 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2001_06 ride_p_18  (cost=0.00..919.00 rows=45000 width=16) (actual time=0.044..5.145 rows=45000 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2001_07 ride_p_19  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.051..7.484 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2001_08 ride_p_20  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.073..6.294 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2001_09 ride_p_21  (cost=0.00..919.00 rows=45000 width=16) (actual time=0.027..5.502 rows=45000 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2001_10 ride_p_22  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.027..5.795 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2001_11 ride_p_23  (cost=0.00..919.00 rows=45000 width=16) (actual time=0.057..6.088 rows=45000 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2001_12 ride_p_24  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.049..6.436 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2002_01 ride_p_25  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.059..7.556 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2002_02 ride_p_26  (cost=0.00..858.00 rows=42000 width=16) (actual time=0.047..6.540 rows=42000 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2002_03 ride_p_27  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.046..5.852 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2002_04 ride_p_28  (cost=0.00..919.00 rows=45000 width=16) (actual time=0.063..6.132 rows=45000 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2002_05 ride_p_29  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.023..7.284 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2002_06 ride_p_30  (cost=0.00..919.00 rows=45000 width=16) (actual time=0.028..5.775 rows=45000 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2002_07 ride_p_31  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.019..5.587 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2002_08 ride_p_32  (cost=0.00..949.50 rows=46500 width=16) (actual time=0.037..5.862 rows=46500 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2002_09 ride_p_33  (cost=0.00..796.00 rows=39000 width=16) (actual time=0.029..5.854 rows=39000 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2002_10 ride_p_34  (cost=0.00..37.75 rows=9 width=16) (actual time=0.030..0.031 rows=0 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2002_11 ride_p_35  (cost=0.00..37.75 rows=9 width=16) (actual time=0.012..0.012 rows=0 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"  ->  Seq Scan on ride_2002_12 ride_p_36  (cost=0.00..37.75 rows=9 width=16) (actual time=0.010..0.010 rows=0 loops=1)"
"        Filter: ((startdate >= '2000-01-01'::date) AND (startdate <= '2003-11-01'::date))"
"Planning Time: 5.953 ms"
"Execution Time: 378.776 ms"
```

Сложный запрос:
```sql
explain analyze select tickets.id, ride_p.id, bus.model from book.tickets 
inner join book.ride_p on tickets.id = ride_p.id
inner join book.bus on ride_p.fkbus = bus.id
where ride_p.startdate between '2001-05-26'::date and '2002-05-26'::date

"Gather  (cost=1001.68..18105.56 rows=13728 width=44) (actual time=0.701..754.700 rows=549000 loops=1)"
"  Workers Planned: 2"
"  Workers Launched: 2"
"  ->  Nested Loop  (cost=1.68..15732.76 rows=5720 width=44) (actual time=1.258..686.271 rows=183000 loops=3)"
"        ->  Hash Join  (cost=1.11..10219.64 rows=5720 width=36) (actual time=1.174..133.933 rows=183000 loops=3)"
"              Hash Cond: (ride_p.fkbus = bus.id)"
"              ->  Parallel Append  (cost=0.00..9605.19 rows=228803 width=8) (actual time=0.923..75.339 rows=183000 loops=3)"
"                    ->  Parallel Seq Scan on ride_2001_05 ride_p_1  (cost=0.00..662.29 rows=5373 width=8) (actual time=2.685..4.991 rows=9000 loops=1)"
"                          Filter: ((startdate >= '2001-05-26'::date) AND (startdate <= '2002-05-26'::date))"
"                          Rows Removed by Filter: 37500"
"                    ->  Parallel Seq Scan on ride_2001_07 ride_p_3  (cost=0.00..662.29 rows=27353 width=8) (actual time=0.067..13.490 rows=46500 loops=1)"
"                          Filter: ((startdate >= '2001-05-26'::date) AND (startdate <= '2002-05-26'::date))"
"                    ->  Parallel Seq Scan on ride_2001_08 ride_p_4  (cost=0.00..662.29 rows=27353 width=8) (actual time=0.027..11.359 rows=46500 loops=1)"
"                          Filter: ((startdate >= '2001-05-26'::date) AND (startdate <= '2002-05-26'::date))"
"                    ->  Parallel Seq Scan on ride_2001_10 ride_p_6  (cost=0.00..662.29 rows=27353 width=8) (actual time=0.031..13.556 rows=46500 loops=1)"
"                          Filter: ((startdate >= '2001-05-26'::date) AND (startdate <= '2002-05-26'::date))"
"                    ->  Parallel Seq Scan on ride_2001_12 ride_p_8  (cost=0.00..662.29 rows=27353 width=8) (actual time=0.061..12.950 rows=46500 loops=1)"
"                          Filter: ((startdate >= '2001-05-26'::date) AND (startdate <= '2002-05-26'::date))"
"                    ->  Parallel Seq Scan on ride_2002_01 ride_p_9  (cost=0.00..662.29 rows=27353 width=8) (actual time=0.023..12.127 rows=46500 loops=1)"
"                          Filter: ((startdate >= '2001-05-26'::date) AND (startdate <= '2002-05-26'::date))"
"                    ->  Parallel Seq Scan on ride_2002_03 ride_p_11  (cost=0.00..662.29 rows=27353 width=8) (actual time=0.036..14.175 rows=46500 loops=1)"
"                          Filter: ((startdate >= '2001-05-26'::date) AND (startdate <= '2002-05-26'::date))"
"                    ->  Parallel Seq Scan on ride_2002_05 ride_p_13  (cost=0.00..662.29 rows=22937 width=8) (actual time=0.015..11.042 rows=39000 loops=1)"
"                          Filter: ((startdate >= '2001-05-26'::date) AND (startdate <= '2002-05-26'::date))"
"                          Rows Removed by Filter: 7500"
"                    ->  Parallel Seq Scan on ride_2001_06 ride_p_2  (cost=0.00..641.06 rows=26471 width=8) (actual time=0.014..10.987 rows=15000 loops=3)"
"                          Filter: ((startdate >= '2001-05-26'::date) AND (startdate <= '2002-05-26'::date))"
"                    ->  Parallel Seq Scan on ride_2001_09 ride_p_5  (cost=0.00..641.06 rows=26471 width=8) (actual time=0.007..12.114 rows=45000 loops=1)"
"                          Filter: ((startdate >= '2001-05-26'::date) AND (startdate <= '2002-05-26'::date))"
"                    ->  Parallel Seq Scan on ride_2001_11 ride_p_7  (cost=0.00..641.06 rows=26471 width=8) (actual time=0.008..11.898 rows=45000 loops=1)"
"                          Filter: ((startdate >= '2001-05-26'::date) AND (startdate <= '2002-05-26'::date))"
"                    ->  Parallel Seq Scan on ride_2002_04 ride_p_12  (cost=0.00..641.06 rows=26471 width=8) (actual time=0.007..11.378 rows=45000 loops=1)"
"                          Filter: ((startdate >= '2001-05-26'::date) AND (startdate <= '2002-05-26'::date))"
"                    ->  Parallel Seq Scan on ride_2002_02 ride_p_10  (cost=0.00..598.59 rows=24706 width=8) (actual time=0.012..11.500 rows=42000 loops=1)"
"                          Filter: ((startdate >= '2001-05-26'::date) AND (startdate <= '2002-05-26'::date))"
"              ->  Hash  (cost=1.05..1.05 rows=5 width=36) (actual time=0.050..0.050 rows=5 loops=3)"
"                    Buckets: 1024  Batches: 1  Memory Usage: 9kB"
"                    ->  Seq Scan on bus  (cost=0.00..1.05 rows=5 width=36) (actual time=0.044..0.045 rows=5 loops=3)"
"        ->  Index Only Scan using tickets_pkey on tickets  (cost=0.56..0.96 rows=1 width=8) (actual time=0.003..0.003 rows=1 loops=549000)"
"              Index Cond: (id = ride_p.id)"
"              Heap Fetches: 0"
"Planning Time: 0.800 ms"
"Execution Time: 780.358 ms"
```

Незначительное улучшение скоротси при партицонировании, при указнии конкретной патриции значительный рост скорости доступа к данным
При достпе к большим таблицам скорость доступа к одной таблице выше чем к партиционированной таблице

### Выделяем два дополнительных диска для wal файлов + для партиционированной таблицы и проверим скорость записи и чтения данных

```bash
/dev/sdc1                        12G  145M   11G   2% /data1
/dev/sdd1                       9.8G   73M  9.2G   1% /data2
```

Создание табличного пространсва и перенос партиций 
```sql
CREATE TABLESPACE tbs1 LOCATION '/data2/tbs1';

ALTER TABLE book.ride_p SET TABLESPACE tbs1;

ALTER TABLE book.ride_2002_10 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2002_09 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2002_08 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2002_06 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2002_05 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2002_04 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2002_03 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2002_02 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2001_12 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2001_11 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2001_08 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2001_07 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2001_06 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2001_05 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2001_04 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2001_03 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2001_02 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2000_11 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2000_09 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2000_08 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2000_07 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2000_06 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2000_05 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2002_07 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2000_04 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2000_03 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2000_02 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2001_10 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2001_09 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2001_01 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2000_12 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2000_10 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2002_01 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2002_11 SET TABLESPACE tbs1;
ALTER TABLE book.ride_2002_12 SET TABLESPACE tbs1;
```

Перенос wal файлов
```bash
mv $PGDATA/pg_wal /data1/pg_wal
ln -s /data1/pg_wal $PGDATA/pg_wal

chown -R postgres:postgres /data1/pg_wal
chmod 700 /data/pg_wal
```

Проверка скорости чтения данных

Партиционированная таблица
```bash
bash-4.4$ pgbench -c 100 -j 4 -T 10 -f ~/select_part.sql -U postgres -h localhost -p 5433 thai -n | grep -E "latency|tps";
latency average = 38.569 ms
tps = 2592.738809 (without initial connection time)
bash-4.4$ pgbench -c 200 -j 4 -T 10 -f ~/select_part.sql -U postgres -h localhost -p 5433 thai -n | grep -E "latency|tps";
latency average = 76.759 ms
tps = 2605.551310 (without initial connection time)
bash-4.4$ pgbench -c 300 -j 4 -T 10 -f ~/select_part.sql -U postgres -h localhost -p 5433 thai -n | grep -E "latency|tps";
latency average = 129.135 ms
tps = 2323.149133 (without initial connection time)
bash-4.4$ pgbench -c 400 -j 4 -T 10 -f ~/select_part.sql -U postgres -h localhost -p 5433 thai -n | grep -E "latency|tps";
latency average = 216.829 ms
tps = 1844.769851 (without initial connection time)
```
Непартиционированная таблица
```bash
bash-4.4$ pgbench -c 100 -j 4 -T 10 -f ~/select_not_part.sql -U postgres -h localhost -p 5433 thai -n | grep -E "latency|tps";
latency average = 4.530 ms
tps = 22072.825695 (without initial connection time)
bash-4.4$ pgbench -c 200 -j 4 -T 10 -f ~/select_not_part.sql -U postgres -h localhost -p 5433 thai -n | grep -E "latency|tps";
latency average = 10.310 ms
tps = 19398.335405 (without initial connection time)
bash-4.4$ pgbench -c 300 -j 4 -T 10 -f ~/select_not_part.sql -U postgres -h localhost -p 5433 thai -n | grep -E "latency|tps";
latency average = 16.252 ms
tps = 18459.291865 (without initial connection time)
bash-4.4$ pgbench -c 400 -j 4 -T 10 -f ~/select_not_part.sql -U postgres -h localhost -p 5433 thai -n | grep -E "latency|tps";
latency average = 23.007 ms
tps = 17385.767980 (without initial connection time)
```
Проверка скорости вставки 

Непартиционированная таблица
```bash
pgbench -c 100 -j 4 -T 10 -f ~/part_insert.sql -U postgres -h localhost -p 5433 thai -n | grep -E "latency|tps";
latency average = 16.185 ms
tps = 6178.393238 (without initial connection time)
pgbench -c 200 -j 4 -T 10 -f ~/part_insert.sql -U postgres -h localhost -p 5433 thai -n | grep -E "latency|tps";
latency average = 27.898 ms
tps = 7168.898182 (without initial connection time)
pgbench -c 300 -j 4 -T 10 -f ~/part_insert.sql -U postgres -h localhost -p 5433 thai -n | grep -E "latency|tps";
latency average = 39.840 ms
tps = 7530.138688 (without initial connection time)
pgbench -c 400 -j 4 -T 10 -f ~/part_insert.sql -U postgres -h localhost -p 5433 thai -n | grep -E "latency|tps";
latency average = 69.793 ms
tps = 5731.203469 (without initial connection time)
```
партиционированная таблица
```bash
pgbench -c 100 -j 4 -T 10 -f ~/part_insert.sql -U postgres -h localhost -p 5433 thai -n | grep -E "latency|tps";
latency average = 12.615 ms
tps = 7927.264846 (without initial connection time)
pgbench -c 200 -j 4 -T 10 -f ~/part_insert.sql -U postgres -h localhost -p 5433 thai -n | grep -E "latency|tps";
latency average = 30.888 ms
tps = 6475.022425 (without initial connection time)
pgbench -c 300 -j 4 -T 10 -f ~/part_insert.sql -U postgres -h localhost -p 5433 thai -n | grep -E "latency|tps";
latency average = 62.144 ms
tps = 4827.492433 (without initial connection time)
pgbench -c 400 -j 4 -T 10 -f ~/part_insert.sql -U postgres -h localhost -p 5433 thai -n | grep -E "latency|tps";
latency average = 109.081 ms
tps = 3667.008069 (without initial connection time)
```

Наблюдения:
 - Партиции удобный инструмент для оперирование 'физически' данными, при удалении освобождается место в ос
 - При выборке всех данных, на моем оборудовании, партиции работают медленее чем из непартиционированной таблицы
 - Массовая вставка и массовое чтение также работает быстрее для непартиционированной таблицы
 
Применяем партиции с осторожностью, если хотим:
 - Часто чистить таблицы
 - Запросы будут уходить только в конктрентую партицию
 - Большая таблица(сотни гигабайт)
 
 