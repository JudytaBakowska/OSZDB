# POSTGRESQL Pierwsze wprawki

Postgres zainstalowany został w kontenerze dockerowym z uyciem pliku dokcer-compose:
```yml
version: "3.8"  # Specify the Docker Compose version

services:
  postgres:
    image: postgres:latest  # Use the official PostgreSQL image
    restart: unless-stopped  # Restart container automatically unless manually stopped
    environment:
      POSTGRES_USER: postgres  # Username for PostgreSQL
    ports:
      - "5432:5432"  # Map the container port 5432 to host port 5432
    volumes:
      - postgres_data:/var/lib/postgresql/data  # Persist data volume

volumes:
  postgres_data: {}  # Define an empty volume for persistent data

```


### Tworzenie instancji serwera w lokalizacji /tmp/test_db 

```bash
docker exec -it —user postgres  <id kontenera> bash #logowanie do kontenera 

#edycja portu na ktorym bedzie wystawiony serwer
vi tmp/test_db/postgresql.conf #port = 5440


pg_ctl -D /tmp/test_db start #uruchomienie instancji
```

### Podstawowe operacje w bazie


Tworzeie tablei tbl o strukturze: int id PK, varchar name
```bash
postgres@postgres1:/$ psql
psql (16.2 (Debian 16.2-1.pgdg120+2))
Type "help" for help.

postgres=# CREATE TABLE tbl (
  id INT PRIMARY KEY,
  name VARCHAR(255)
);
CREATE TABLE
```


Wyświetl schemat tabeli tbl,  wstaw do tabeli tbl kilka przykładowych rekordów
```bash
postgres@postgres1:/$ psql -U postgres -c "INSERT INTO tbl (id, name) VALUES (1, 'Eve'), (2,'Adam'), (3, 'John'), (4, 'Julie');"
INSERT 0 4

postgres@postgres1:/$ psql -U postgres -c 'select * from tbl'
 id | name  
----+-------
  1 | Eve
  2 | Adam
  3 | John
  4 | Julie
(4 rows)
```

Usuń rekordy o id mniejszym od 3
``` bash
root@postgres1:/$ psql -U postgres -c "DELETE from tbl WHERE id < 3"
DELETE 2

```

Usuń z tabeli wszystkie rekordy
``` bash
root@postgres1:/$ psql -U postgres -c "DELETE from tbl"
DELETE 2
```
Usuń tabele tbl;
``` bash
root@postgres1:/$ psql -U postgres -c "DROP TABLE tbl"
DROP TABLE
```

Zamknij sesje
``` bash
ctrl+D
```
Zatrzymaj instancje test_db
``` bash
postgres@postgres1:/$ pg_ctl -D /tmp/test_db stop
waiting for server to shut down...2024-05-06 13:14:53.081 UTC [85] LOG:  received fast shutdown request
.2024-05-06 13:14:53.086 UTC [85] LOG:  aborting any active transactions
2024-05-06 13:14:53.089 UTC [85] LOG:  background worker "logical replication launcher" (PID 91) exited with exit code 1
2024-05-06 13:14:53.089 UTC [86] LOG:  shutting down
2024-05-06 13:14:53.093 UTC [86] LOG:  checkpoint starting: shutdown immediate
2024-05-06 13:14:53.108 UTC [86] LOG:  checkpoint complete: wrote 0 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.001 s, sync=0.001 s, total=0.019 s; sync files=0, longest=0.000 s, average=0.000 s; distance=0 kB, estimate=0 kB; lsn=0/152B4E8, redo lsn=0/152B4E8
2024-05-06 13:14:53.112 UTC [85] LOG:  database system is shut down
 done
server stopped
```



# Replikacja strumieniowa

## Stworzenie repliki

Stworzenie nowej instancji serwera
``` bash
postgres@postgres1:/$ initdb -D tmp/primary_db
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.utf8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

creating directory tmp/primary_db ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok

initdb: warning: enabling "trust" authentication for local connections
initdb: hint: You can change this by editing pg_hba.conf or using the option -A, or --auth-local and --auth-host, the next time you run initdb.

Success. You can now start the database server using:

    pg_ctl -D tmp/primary_db -l logfile start
```

Konfiguracja instancji
``` bash

```
``` bash
postgres@postgres1:/$ pg_ctl -D /tmp/primary_db start
waiting for server to start....2024-05-06 13:24:31.933 UTC [123] LOG:  starting PostgreSQL 16.2 (Debian 16.2-1.pgdg120+2) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14) 12.2.0, 64-bit
2024-05-06 13:24:31.933 UTC [123] LOG:  listening on IPv4 address "0.0.0.0", port 5433
2024-05-06 13:24:31.933 UTC [123] LOG:  listening on IPv6 address "::", port 5433
2024-05-06 13:24:31.942 UTC [123] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5433"
2024-05-06 13:24:31.953 UTC [126] LOG:  database system was shut down at 2024-05-06 13:19:38 UTC
2024-05-06 13:24:31.961 UTC [123] LOG:  database system is ready to accept connections
 done
server started
```


Utworzenie usera `repuser`
``` bash
postgres@postgres1:/$ psql -U postgres -p 5433
psql (16.2 (Debian 16.2-1.pgdg120+2))
Type "help" for help.

postgres=# CREATE USER repuser replication;
CREATE ROLE
postgres=# \du
                             List of roles
 Role name |                         Attributes                         
-----------+------------------------------------------------------------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS
 repuser   | Replication
```


Dodanie moliwości łączenia się do bazy uytkownikowi repuser
``` bash
postgres=# CREATE USER repuser replication;
CREATE ROLE
postgres=# \du
                             List of roles
 Role name |                         Attributes                         
-----------+------------------------------------------------------------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS
 repuser   | Replication
```

Restart instancji

``` bash
postgres@postgres1:/$ pg_ctl -D /tmp/primary_db restart -l tmp/primary_db/logfile 
waiting for server to shut down.... done
server stopped
waiting for server to start.... done
server started
```


Przygotowanie instancji repliki
``` bash
postgres@postgres1:/$ pg_basebackup -h localhost --user repuser -p 5433 -D /tmp/replica_db -R -C --slot=slot_name --checkpoint=fast 


postgres@postgres1:/$ ls -al tmp/replica_db/
total 276
drwx------ 19 postgres postgres   4096 May  6 13:42 .
drwxrwxrwt  1 root     root       4096 May  6 13:42 ..
-rw-------  1 postgres postgres    225 May  6 13:42 backup_label
-rw-------  1 postgres postgres 137461 May  6 13:42 backup_manifest
drwx------  5 postgres postgres   4096 May  6 13:42 base
drwx------  2 postgres postgres   4096 May  6 13:42 global
-rw-r--r--  1 postgres postgres   5225 May  6 13:42 logfile
drwx------  2 postgres postgres   4096 May  6 13:42 pg_commit_ts
drwx------  2 postgres postgres   4096 May  6 13:42 pg_dynshmem
-rw-------  1 postgres postgres   5747 May  6 13:42 pg_hba.conf
-rw-------  1 postgres postgres   2640 May  6 13:42 pg_ident.conf
drwx------  4 postgres postgres   4096 May  6 13:42 pg_logical
drwx------  4 postgres postgres   4096 May  6 13:42 pg_multixact
drwx------  2 postgres postgres   4096 May  6 13:42 pg_notify
drwx------  2 postgres postgres   4096 May  6 13:42 pg_replslot
drwx------  2 postgres postgres   4096 May  6 13:42 pg_serial
drwx------  2 postgres postgres   4096 May  6 13:42 pg_snapshots
drwx------  2 postgres postgres   4096 May  6 13:42 pg_stat
drwx------  2 postgres postgres   4096 May  6 13:42 pg_stat_tmp
drwx------  2 postgres postgres   4096 May  6 13:42 pg_subtrans
drwx------  2 postgres postgres   4096 May  6 13:42 pg_tblspc
drwx------  2 postgres postgres   4096 May  6 13:42 pg_twophase
-rw-------  1 postgres postgres      3 May  6 13:42 PG_VERSION
drwx------  3 postgres postgres   4096 May  6 13:42 pg_wal
drwx------  2 postgres postgres   4096 May  6 13:42 pg_xact
-rw-------  1 postgres postgres    441 May  6 13:42 postgresql.auto.conf
-rw-------  1 postgres postgres  29769 May  6 13:42 postgresql.conf
-rw-------  1 postgres postgres      0 May  6 13:42 standby.signal
```

Zmiana portu repliki na 5434 i uruchomienie repliki
```bash
postgres@postgres1:/$ pg_ctl -D /tmp/replica_db start -l tmp/replica_db/logfile 
waiting for server to start.... done
server started


## replica_db/logfile
2024-05-06 13:45:10.960 UTC [255] LOG:  listening on IPv4 address "0.0.0.0", port 5434
2024-05-06 13:45:10.960 UTC [255] LOG:  listening on IPv6 address "::", port 5434
2024-05-06 13:45:10.970 UTC [255] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5434"
2024-05-06 13:45:10.982 UTC [258] LOG:  database system was interrupted; last known up at 2024-05-06 13:42:21 UTC
2024-05-06 13:45:11.146 UTC [258] LOG:  entering standby mode
2024-05-06 13:45:11.147 UTC [258] LOG:  starting backup recovery with redo LSN 0/2000028, checkpoint LSN 0/2000060, on timeline ID 1
2024-05-06 13:45:11.153 UTC [258] LOG:  redo starts at 0/2000028
2024-05-06 13:45:11.155 UTC [258] LOG:  completed backup recovery with redo LSN 0/2000028 and end LSN 0/2000138
2024-05-06 13:45:11.155 UTC [258] LOG:  consistent recovery state reached at 0/2000138
2024-05-06 13:45:11.155 UTC [255] LOG:  database system is ready to accept read-only connections
2024-05-06 13:45:11.170 UTC [259] LOG:  started streaming WAL from primary at 0/3000000 on timeline 1
```


Widok `pg_stat_replication` w instancji `primary`
``` sql 
SELECT * FROM  pg_stat_replication;
```

``` bash
 pid | usesysid | usename | application_name | client_addr | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_lsn  | write_lsn | flush_lsn | replay_lsn | write_lag | flush_lag | replay_lag | sync_priority | sync_state |          reply_time           
-----+----------+---------+------------------+-------------+-----------------+-------------+-------------------------------+--------------+-----------+-----------+-----------+-----------+------------+-----------+-----------+------------+---------------+------------+-------------------------------
 260 |    16388 | repuser | walreceiver      | 127.0.0.1   | localhost       |       44976 | 2024-05-06 13:45:11.167099+00 |              | streaming | 0/3000148 | 0/3000148 | 0/3000148 | 0/3000148  |           |           |            |             0 | async      | 2024-05-06 13:48:28.465618+00
(1 row)
```

Widok pg_stat_wal_receiver

``` bash

 pid |  status   | receive_start_lsn | receive_start_tli | written_lsn | flushed_lsn | received_tli |      last_msg_send_time       |     last_msg_receipt_time     | latest_end_lsn |        latest_end_time        | slot_name | sender_host | sender_port |                                                                                                                                                                             conninfo                                                                                                                                                                             
-----+-----------+-------------------+-------------------+-------------+-------------+--------------+-------------------------------+-------------------------------+----------------+-------------------------------+-----------+-------------+-------------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 259 | streaming | 0/3000000         |                 1 | 0/3000148   | 0/3000148   |            1 | 2024-05-06 13:51:58.621869+00 | 2024-05-06 13:51:58.621995+00 | 0/3000148      | 2024-05-06 13:47:28.440691+00 | slot_name | localhost   |        5433 | user=repuser passfile=/var/lib/postgresql/.pgpass channel_binding=prefer dbname=replication host=localhost port=5433 fallback_application_name=walreceiver sslmode=prefer sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable
(1 row)
```

## Testowanie replikacji

### Test replikacji nowych tabel i rekordów

Stworzenie tabeli w bazie `primary_db` ( port 5433 ) i dodanie rekordów. 
``` bash
postgres@postgres1:/$ psql -p 5433 -c "CREATE TABLE new_table ( id INT PRIMARY KEY, value VARCHAR(50));"
CREATE TABLE


postgres@postgres1:/$ psql -p 5433  -c "INSERT INTO new_table (id, value) VALUES (1, 'Eve'), (2,'Adam'), (3, 'John'), (4, 'Julie');"
INSERT 0 4

```
Prwadzenie replikacji tabeli do bazy `replica_db` ( port 5434 )
``` bash
postgres@postgres1:/$ psql -p 5434 -c "SELECT * FROM new_table;"
 id | value 
----+-------
  1 | Eve
  2 | Adam
  3 | John
  4 | Julie
(4 rows)
```

### Test replikacji operacji DELETE i TRUNCATE

Usunięcie rekordu o `id` równym 3 z bazy `primary`
``` bash
postgres@postgres1:/$ psql -p 5433 -c "DELETE FROM new_table WHERE id=3"
DELETE 1

```

Replikacja operacji w bazie `replica`

``` bash
postgres@postgres1:/$ psql -p 5434 -c "SELECT * FROM new_table;"
 id | value 
----+-------
  1 | Eve
  2 | Adam
  4 | Julie
(3 rows)
```

Usunięcie rekordów w bazie `primary` za pomocą operacji `TRUNCATE`
```bash
postgres@postgres1:/$ psql -p 5433 -c "TRUNCATE TABLE new_table;"
TRUNCATE TABLE
```
Replikacja operacji w bazie `replica`

``` bash
postgres@postgres1:/$ psql -p 5434 -c "SELECT * FROM new_table;"
 id | value 
----+-------
(0 rows)
```


### Symulacja awarii instancji `primary`

Zasymulowanie awarii przez zastopowanie serwera `primary`
``` bash
postgres@postgres1:/$ pg_ctl -D /tmp/primary_db stop
waiting for server to shut down.... done
server stopped

## primary_db/logfile

2024-05-06 14:11:47.322 UTC [207] LOG:  aborting any active transactions
2024-05-06 14:11:47.325 UTC [207] LOG:  background worker "logical replication launcher" (PID 213) exited with exit code 1
2024-05-06 14:11:47.325 UTC [208] LOG:  shutting down
2024-05-06 14:11:47.332 UTC [208] LOG:  checkpoint starting: shutdown immediate
2024-05-06 14:11:47.355 UTC [208] LOG:  checkpoint complete: wrote 0 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.001 s, sync=0.001 s, total=0.030 s; sync files=0, longest=0.000 s, average=0.000 s; distance=0 kB, estimate=10761 kB; lsn=0/3028028, redo lsn=0/3028028
2024-05-06 14:11:47.365 UTC [207] LOG:  database system is shut down
```

Wyznaczenie serwera `replica` jako głównej instancji
``` bash
postgres@postgres1:/$ psql -p 5434 -c "SELECT pg_promote();"
 pg_promote 
------------
 t
(1 row)

```


W logach instancji `replica_db` widoczny jest komunikat `received promote request` oznaczający mianowanie tej instancji do roli serwera podstawowego.

``` bash
2024-05-06 14:14:17.499 UTC [258] LOG:  waiting for WAL to become available at 0/30280B8
2024-05-06 14:14:22.021 UTC [258] LOG:  received promote request
2024-05-06 14:14:22.021 UTC [258] LOG:  redo done at 0/3028028 system usage: CPU: user: 0.01 s, system: 0.04 s, elapsed: 1750.86 s
2024-05-06 14:14:22.021 UTC [258] LOG:  last completed transaction was at log time 2024-05-06 14:06:22.590183+00
2024-05-06 14:14:22.031 UTC [258] LOG:  selected new timeline ID: 2
2024-05-06 14:14:22.177 UTC [258] LOG:  archive recovery complete
2024-05-06 14:14:22.190 UTC [256] LOG:  checkpoint starting: force
2024-05-06 14:14:22.195 UTC [255] LOG:  database system is ready to accept connections
2024-05-06 14:14:22.228 UTC [256] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.009 s, sync=0.004 s, total=0.038 s; sync files=2, longest=0.002 s, average=0.002 s; distance=0 kB, estimate=9685 kB; lsn=0/3028108, redo lsn=0/30280D0
```


`Replica_db` nie jest juz jedynie w stanie `read-only`, mozliwe jest wstawianie nowych rekordów. 

``` bash
postgres@postgres1:/$ psql -p 5434 -c "INSERT INTO new_table (id, value) VALUES (1, 'Value');"
INSERT 0 1
```




