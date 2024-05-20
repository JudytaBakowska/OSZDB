Judyta Bąkowska, Karolina Źróbek
# PgSQL – Replikacja logiczna

### Setup instancji *publisher* i *subscriber*
Utworznie z poziomu użytkownika postgres nowego klastera postgresowego w lokalizacji `/tmp/publisher_db`.

``` bash
postgres@postgres1:/$ initdb /tmp/publisher_db
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.utf8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

creating directory /tmp/publisher_db ... ok
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

    pg_ctl -D /tmp/publisher_db -l logfile start
```



Utworznie z poziomu użytkownika postgres nowego klastera postgresowego w lokalizacji
`/tmp/subscriber_db`.

``` bash
postgres@postgres1:/$ initdb /tmp/subscriber_db
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.utf8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

creating directory /tmp/subscriber_db ... ok
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

    pg_ctl -D /tmp/subscriber_db -l logfile start
```


Zmiany w konfiguracji *publishera*: ustawienie `wal_level`na `logical`  i portu, na którym instancja ta będzie uruchomiona na 5433.

``` bash
#---------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#----------------------------------------------------------

# - Connection Settings -

listen_addresses = '*'
	# comma-separated list of addresses;
	# defaults to 'localhost'; use '*' for all
	# (change requires restart)
port = 5433	# (change requires restart)

...

#---------------------------------------------------------
# WRITE-AHEAD LOG
#---------------------------------------------------------

# - Settings -

wal_level = logical	# minimal, replica, or logical
			# (change requires restart)

```

Zmiana portu subscribera na 5434.

```bash
#---------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#----------------------------------------------------------

# - Connection Settings -

listen_addresses = '*'
	# comma-separated list of addresses;
	# defaults to 'localhost'; use '*' for all
	# (change requires restart)
port = 5434	# (change requires restart)

```

Uruchomienie obu instancji
``` bash
postgres@postgres1:/$ touch tmp/subscriber_db/logfile
postgres@postgres1:/$ touch tmp/publisher_db/logfile

postgres@postgres1:/$ pg_ctl -D /tmp/publisher_db -l /tmp/publisher_db/logfile start
waiting for server to start.... done
server started
postgres@postgres1:/$ pg_ctl -D /tmp/subscriber_db -l /tmp/subscriber_db/logfile start
waiting for server to start.... done
server started

```


Stworzenie bazy danych w instancji `publisher` i dodanie przykładowej tabeli oraz rekordów.


``` bash
postgres@postgres1:/$ psql -p 5433 -c "CREATE DATABASE
pub_db;"
CREATE DATABASE


postgres@postgres1:/$ psql -p 5433 -d pub_db -a -f 
tmp/create_pub_tbl 

```

``` sql
--  Zawartosc pliku /tmp/create_pub_tbl
CREATE TABLE pub_tbl(
  id INT PRIMARY KEY,
  name VARCHAR(255)
);

INSERT INTO pub_tbl (id, name)
SELECT 
    i AS id,
    ('Name ' || i) AS name
FROM 
    generate_series(1, 10) AS s(i);

```

```bash
postgres@postgres1:/$ psql -p 5433 -c "SELECT * FROM
pub_tbl"
 id |  name   
----+---------
  1 | Name 1
  2 | Name 2
  3 | Name 3
  4 | Name 4
  5 | Name 5
  6 | Name 6
  7 | Name 7
  8 | Name 8
  9 | Name 9
 10 | Name 10
(10 rows)

```

Utworzenie bazy `sub_db` w instancji `subscriber`.
``` bash
postgres@postgres1:/$ psql -p 5434 -c "CREATE DATABASE
sub_db;"
CREATE DATABASE
```

Wykonanie zrzutu schematu tabeli `pub_tbl` do pliku z wykorzystaniem polecenia `pg_dump`.
``` bash
postgres@postgres1:/$ psql -p 5433
postgres=\# pg_dump --schema-only -t pub_tbl pub_db > tmp/tbl_structure_dump.sql;
```

Załadowanie schematu z pliku w instancji `subscriber`. 
``` bash
postgres@postgres1:/$ psql -f /tmp/tbl_structure_dump.sql
-p 5434 sub_db
SET
SET
SET
SET
SET
 set_config 
------------
 
(1 row)

SET
SET
SET
SET
SET
SET
CREATE TABLE
ALTER TABLE
ALTER TABLE
```
Wyświetlenie nowozaładowanego schematu tabeli w instancji `subscriber`.
``` bash
postgres@postgres1:/$ psql -p 5434
psql (16.2 (Debian 16.2-1.pgdg120+2))
Type "help" for help.

postgres=# \c sub_db
You are now connected to database "sub_db" as 
user "postgres".

sub_db=# \d pub_tbl
                      Table "public.pub_tbl"
 Column |          Type          | Collation | Nullable | Default 
--------+------------------------+-----------+----------+---------
 id     | integer                |           | not null | 
 name   | character varying(255) |           |          | 
Indexes:
    "pub_tbl_pkey" PRIMARY KEY, btree (id)
```

### Stworzenie *publikacji* i *subskrypcji* 

Utworzenie publikacji za pomocą polecenia `CREATE PUBLICATION`.
``` bash
postgres@postgres1:/$ psql -p 5433
psql (16.2 (Debian 16.2-1.pgdg120+2))
Type "help" for help.

postgres=# CREATE PUBLICATION publication FOR TABLE pub_tbl;
CREATE PUBLICATION
```

Utworzenie subskrypcji z użyciem `CREATE SUBSCRIPTION`.
``` bash
postgres@postgres1:/$ psql -p 5434
psql (16.2 (Debian 16.2-1.pgdg120+2))
Type "help" for help.

postgres=\# \c sub_db
You are now connected to database "sub_db" as user "postgres".
sub_db=\# CREATE SUBSCRIPTION my_subscription 
CONNECTION 'host=localhost port=5433 user=postgres dbname=pub_db' 
PUBLICATION my_publication;
2024-05-20 14:10:01.573 UTC [294] LOG:  logical decoding 
found consistent point at 0/19BEB58
2024-05-20 14:10:01.573 UTC [294] DETAIL:  There are no 
running transactions.
2024-05-20 14:10:01.573 UTC [294] STATEMENT:  
CREATE_REPLICATION_SLOT "my_subscription" 
LOGICAL pgoutput (SNAPSHOT 'nothing')
NOTICE:  created replication slot "my_subscription" 
on publisher
CREATE SUBSCRIPTION

sub_db=\# 2024-05-20 14:10:01.591 UTC [295] LOG:  logical replication 
apply worker for subscription "my_subscription" has started
2024-05-20 14:10:01.599 UTC [296] LOG:  starting logical decoding for 
slot "my_subscription"
2024-05-20 14:10:01.599 UTC [296] DETAIL:  Streaming transactions 
committing after 0/19BEB90, reading WAL from 0/19BEB58.
2024-05-20 14:10:01.599 UTC [296] STATEMENT:  START_REPLICATION SLOT 
"my_subscription" LOGICAL 0/0 (proto_version '4', origin 'any', publication_names '"my_publication"')
2024-05-20 14:10:01.600 UTC [296] LOG:  logical decoding found consistent 
point at 0/19BEB58
2024-05-20 14:10:01.600 UTC [296] DETAIL:  There are no running 
transactions.
2024-05-20 14:10:01.600 UTC [296] STATEMENT:  START_REPLICATION SLOT 
"my_subscription" LOGICAL 0/0 (proto_version '4', origin 'any', publication_names '"my_publication"')
2024-05-20 14:10:01.605 UTC [297] LOG:  logical replication table 
synchronization worker for subscription "my_subscription", table "pub_tbl" has started
2024-05-20 14:10:01.654 UTC [298] LOG:  logical decoding found 
consistent point at 0/19BEB90
2024-05-20 14:10:01.654 UTC [298] DETAIL:  There are no running 
transactions.
2024-05-20 14:10:01.654 UTC [298] STATEMENT:  CREATE_REPLICATION_SLOT 
"pg_24581_sync_16389_7368474158716088419" LOGICAL pgoutput (SNAPSHOT 'use')
2024-05-20 14:10:01.684 UTC [298] LOG:  starting logical decoding 
for slot "pg_24581_sync_16389_7368474158716088419"
2024-05-20 14:10:01.684 UTC [298] DETAIL:  Streaming transactions 
committing after 0/19BEBC8, reading WAL from 0/19BEB90.
2024-05-20 14:10:01.684 UTC [298] STATEMENT:  START_REPLICATION SLOT 
pg_24581_sync_16389_7368474158716088419" LOGICAL 0/19BEBC8 (proto_version
 '4', origin 'any', publication_names '"my_publication"')
2024-05-20 14:10:01.684 UTC [298] LOG:  logical decoding found 
consistent point at 0/19BEB90
2024-05-20 14:10:01.684 UTC [298] DETAIL:  There are no running 
transactions.
2024-05-20 14:10:01.684 UTC [298] STATEMENT:  START_REPLICATION SLOT 
"pg_24581_sync_16389_7368474158716088419" LOGICAL 0/19BEBC8 
(proto_version '4', origin 'any', publication_names '"my_publication"')
2024-05-20 14:10:01.690 UTC [297] LOG:  logical replication 
table synchronization worker for subscription "my_subscription", 
table "pub_tbl" has finished

```
Dotychczas przechowywane w tablei `pub_db.pub_tbl` wiersze zostały zreplikowane.
``` bash
sub_db=# SELECT * FROM pub_tbl;
 id |  name   
----+---------
  1 | Name 1
  2 | Name 2
  3 | Name 3
  4 | Name 4
  5 | Name 5
  6 | Name 6
  7 | Name 7
  8 | Name 8
  9 | Name 9
 10 | Name 10
(10 rows)
```


### Testowanie działania replikacji

``` bash
##Publisher
pub_db=\# INSERT INTO pub_tbl (id, name)
SELECT 
    i AS id,
    ('Name ' || i) AS name FROM generate_series(11, 20) AS s(i);
INSERT 0 10

pub_db=\# SELECT * FROM pub_tbl;
 id |  name   
----+---------
  1 | Name 1
  2 | Name 2
  3 | Name 3
  4 | Name 4
  5 | Name 5
  6 | Name 6
  7 | Name 7
  8 | Name 8
  9 | Name 9
 10 | Name 10
 11 | Name 11
 12 | Name 12
 13 | Name 13
 14 | Name 14
 15 | Name 15
 16 | Name 16
 17 | Name 17
 18 | Name 18
 19 | Name 19
 20 | Name 20
(20 rows)


##Subscriber
sub_db=\# SELECT * FROM pub_tbl;
 id |  name   
----+---------
  1 | Name 1
  2 | Name 2
  3 | Name 3
  4 | Name 4
  5 | Name 5
  6 | Name 6
  7 | Name 7
  8 | Name 8
  9 | Name 9
 10 | Name 10
 11 | Name 11
 12 | Name 12
 13 | Name 13
 14 | Name 14
 15 | Name 15
 16 | Name 16
 17 | Name 17
 18 | Name 18
 19 | Name 19
 20 | Name 20
(20 rows)

```



