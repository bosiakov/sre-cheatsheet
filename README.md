# Reliability Engineer Cheatsheet

Collection of snippets and links for host management: app and db status check, db management and app debugging.

## Check host state in 60s

Memory:

```bash
free -m # memory usage
```

CPU:

```bash
uptime # check load avg
vmstat 1 # system-wide utilization. is this swapping?
mpstat -P ALL 1 # CPU balance
pidstat 1 # per-process CPU
```

Disk:

```bash
iostat -xnz 1 # any disk IO? no? good
df -h # are file systems nearly full?
```

Network:

```bash
sar -n DEV 1 # network IO
sar -n TCP,ETCP 1 # TCP stats
```

Check DB connections:

```bash
lsof -ni TCP:3306 | awk '{print $2}' | uniq -c
# 3306 - default MYSQL port
# 5432 - default PSQL port
# 6432 - default pgBouncer port
```

## Storage

Database actual isolation levels: https://github.com/ept/hermitage

### Postgres

#### Postgres Guides

Postgres Guide : http://postgresguide.com/

Postgres Config generator: https://pgtune.leopard.in.ua/#/

#### Explain visialize

Postgres Explain Beatify: https://explain.depesz.com/

Postgres Explain Visualizer: https://tatiyants.com/pev/#/plans/new

#### Disk usage

Detailted PostGres Disk Usage: https://wiki.postgresql.org/wiki/Disk_Usage

Check DB sizes:

```sql
\l+

-- size of one table 
\c dbname
SELECT pg_size_pretty( pg_total_relation_size('tablename'));
```

#### Running queries

```sql
-- show queries that runs more than 2 minutes
SELECT now() - query_start as "runtime", usename, datname, state, query
  FROM  pg_stat_activity
  WHERE now() - query_start > '2 minutes'::interval
 ORDER BY runtime DESC;

-- show all running queries
SELECT pid, age(clock_timestamp(), query_start), usename, query 
FROM pg_stat_activity 
WHERE query != '<IDLE>' AND query NOT ILIKE '%pg_stat_activity%' 
ORDER BY query_start desc;
```

Check the number of connections:

```
SELECT * FROM pg_stat_activity;
```

Check pgBouncer state:

```sql
psql -p 6432 pgbouncer # Only users listed in the configuration parameters admin_users or stats_users are allowed to log in
SHOW STATS
SHOW TOTALS # is there any anomalies across stats?
```

#### Postgres Dump / Backup

Official Postgres script for automated backups: http://wiki.postgresql.org/wiki/Automated_Backup_on_Linux

Easier way: https://www.depesz.com/2013/09/11/how-to-make-backups-of-postgresql/

Quick snippet for PG backup:

```bash
# dump whole db
pg_dump -Fc --schema='public' --exclude-table='excl.table' -h <host> -U <user> -W -v <db> > db.dump.$(date +%F) # dump a database into a custom-format archive

# dump one table with plane sql
pg_dump --host <host> --port 5432 --username <user> --format plain --verbose --file "/tmp/dump.table.sql" --table public.<table> <db_name>

pg_restore -d newdb db.dump # reload an archive file into a (freshly created) database named newdb
```


### MySQL

#### Check MySQL state in 60s

This query will list the size of every table in every database, largest first:

```
SELECT 
     table_schema as `Database`, 
     table_name AS `Table`, 
     round(((data_length + index_length) / 1024 / 1024), 2) `Size in MB` 
FROM information_schema.TABLES 
ORDER BY (data_length + index_length) DESC;
```

Server Status, connections and running queries:

```
show global status;

show status where `variable_name` = 'Threads_connected'; -- The number of currently open connections.

show processlist; -- running queries
```

#### MySQL dump / backup

Quick snippet for MySQL backup:

```
mysqldump -u <user> -p <password> -h <host> --quick --single-transaction --databases db1 --result-file=db-backup-$(date +%F).sql 
# option --single-transaction sets the transaction isolation mode to REPEATABLE READ and sends a START TRANSACTION SQL statement to the server before dumping data 
mysql db1 < dump.sql
```

### Redis

CLI handbook: https://redis.io/commands

### Check Redis state

```bash
redis-cli info # Complete info including network, CPU, memory, persistence and cluster settings
redis-cli monitor # Warning: Because MONITOR streams back all commands, its use comes at a cost

redis-cli client list # returns information and statistics about the client connections
redis-cli dbsize # the number of keys in the currently-selected database. New connections always use the database 0.
```

## Development

### Helpers

Online generators:

Curl <-> Go https://mholt.github.io/curl-to-go/ https://mholt.github.io/json-to-go/

JSON <-> Any language https://quicktype.io

CURL -> Python/PHP/JSON/etc https://curl.trillworks.com

### Bash

Shell script static analysis tool: https://www.shellcheck.net/

Explain bash string: https://explainshell.com/

How to write good bash: https://blog.yossarian.net/2020/01/23/Anybody-can-write-good-bash-with-a-little-effort

Unofficial bash strict mode: http://redsymbol.net/articles/unofficial-bash-strict-mode/


## Simple perfomance test

Simple example with [wrk](https://github.com/wg/wrk):

```bash
wrk -t12 -c400 -d30s http://127.0.0.1:8080/index.html
```

More advanced example with [hey](https://github.com/rakyll/hey) (written in Go). POST Query with cookies:

```bash
hey -c 200 -z 60s -m POST -T "application/json" -H "Cookie: cookie1=val1" -H "Content-Type: application/json"  -d '{"key1": "val1"}' http://localhost:8000/endpoint
```

Redis benchmark (https://redis.io/topics/benchmarks):

```
redis-benchmark -c 10 -n 100000 -q
```