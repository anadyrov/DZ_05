# DZ_05

-- создать GCE инстанс типа e2-medium и standard disk 10GB
-- ВМ машинка создана 
gcloud compute instances create postgres25032022 --project=melodic-realm-343102 --zone=us-central1-a --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --service-account=337881949322-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=postgres25032022,image=projects/ubuntu-os-cloud/global/images/ubuntu-1804-bionic-v20220308,mode=rw,size=10,type=projects/melodic-realm-343102/zones/us-central1-a/diskTypes/pd-ssd --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
Created [https://www.googleapis.com/compute/v1/projects/melodic-realm-343102/zones/us-central1-a/instances/postgres25032022].
NAME: postgres25032022
ZONE: us-central1-a
MACHINE_TYPE: e2-medium
PREEMPTIBLE:
INTERNAL_IP: 10.128.0.11
EXTERNAL_IP: 34.136.141.80
STATUS: RUNNING

-- установить на него PostgreSQL 13 с дефолтными настройками
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-13

postgres@postgres25032022:/home/anadyrov$ psql 
psql (13.6 (Ubuntu 13.6-1.pgdg18.04+1))
Type "help" for help.

postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges   
-----------+----------+----------+---------+---------+-----------------------
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 | 
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
(3 rows)

-- применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
-- Выставил параметры и перегрузил кластер

ALTER SYSTEM SET max_connections = '40';
ALTER SYSTEM SET shared_buffers = '1GB';
ALTER SYSTEM SET effective_cache_size = '3GB';
ALTER SYSTEM SET maintenance_work_mem = '512MB';
ALTER SYSTEM SET checkpoint_completion_target = '0.9';
ALTER SYSTEM SET wal_buffers = '16MB';
ALTER SYSTEM SET default_statistics_target = '500';
ALTER SYSTEM SET random_page_cost = '4';
ALTER SYSTEM SET effective_io_concurrency = '2';
ALTER SYSTEM SET work_mem = '6553kB';
ALTER SYSTEM SET min_wal_size = '4GB';
ALTER SYSTEM SET max_wal_size = '16GB';

pg_ctlcluster 13 main restart

-- выполнить pgbench -i postgres
-- Запускаем 

postgres@postgres25032022:/home/anadyrov$ pgbench -i postgres
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.10 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.49 s (drop tables 0.02 s, create tables 0.01 s, client-side generate 0.26 s, vacuum 0.12 s, primary keys 0.08 s).

-- запустить pgbench -c8 -P 60 -T 3600 -U postgres postgres
-- дать отработать до конца

postgres@postgres25032022:/home/anadyrov$ pgbench -c8 -P 60 -T 3600 -U postgres postgres
starting vacuum...end.
progress: 60.0 s, 826.9 tps, lat 9.667 ms stddev 5.230
progress: 120.0 s, 822.8 tps, lat 9.723 ms stddev 5.186
progress: 180.0 s, 836.1 tps, lat 9.567 ms stddev 4.934
progress: 240.0 s, 848.3 tps, lat 9.430 ms stddev 4.950
progress: 300.0 s, 835.7 tps, lat 9.572 ms stddev 5.855

Среднее значение ~ 836

-- зафиксировать среднее значение tps в последней ⅙ части работы а дальше настроить autovacuum максимально эффективно
-- так чтобы получить максимально ровное значение tps на горизонте часа
postgres@postgres25032022:/home/anadyrov$ pgbench -c8 -P 60 -T 3600 -U postgres postgres

На мой взгляд удалось немного выровнить? ))

starting vacuum...end.
progress: 60.0 s, 823.3 tps, lat 9.709 ms stddev 6.339
progress: 120.1 s, 654.6 tps, lat 12.201 ms stddev 17.522
progress: 180.1 s, 499.5 tps, lat 16.009 ms stddev 25.066
progress: 240.1 s, 505.2 tps, lat 15.835 ms stddev 25.683
progress: 300.1 s, 503.3 tps, lat 15.889 ms stddev 25.736
progress: 360.1 s, 509.9 tps, lat 15.691 ms stddev 24.959
progress: 420.1 s, 510.8 tps, lat 15.655 ms stddev 24.979
progress: 480.1 s, 490.0 tps, lat 16.326 ms stddev 26.395
progress: 540.1 s, 497.0 tps, lat 16.097 ms stddev 26.133
progress: 600.1 s, 504.7 tps, lat 15.850 ms stddev 25.701
progress: 660.1 s, 503.4 tps, lat 15.890 ms stddev 25.377
progress: 720.1 s, 502.1 tps, lat 15.934 ms stddev 25.613
progress: 780.1 s, 494.1 tps, lat 16.188 ms stddev 26.052
progress: 840.1 s, 492.9 tps, lat 16.228 ms stddev 25.658
progress: 900.1 s, 494.3 tps, lat 16.181 ms stddev 26.197
progress: 960.1 s, 499.7 tps, lat 16.006 ms stddev 25.908

Выставленные параметры:
ALTER SYSTEM SET log_autovacuum_min_duration = 0;
ALTER SYSTEM SET autovacuum_max_workers = 12;
ALTER SYSTEM SET autovacuum_naptime = 12s;
ALTER SYSTEM SET autovacuum_vacuum_threshold = 25;
ALTER SYSTEM SET autovacuum_vacuum_scale_factor = 0.07;
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = 10;
ALTER SYSTEM SET autovacuum_vacuum_cost_limit = 800;





