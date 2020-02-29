# Reliability Engineer Cheatsheet

Collection of snippets and links for host management: app and db status check, db management and app debugging.

## Host

### Check host state in 60s

Memory:

```bash
free -m # memory usage
```
CPU:

```bash
uptime # check load avg
vmstat 1 # system-wide utilization
mpstat -P ALL 1 # CPU balance
pidstat 1 # per-process CPU
```

Disk:

```bash
iostat -xnz 1 # any disk IO? no? good
vmstat 1 # is this swapping?
df -h # are file systems nearly full?
```

Network:

```bash
sar -n DEV 1 # network IO
sar -n TCP,ETCP 1 # TCP stats
```

## DB

Postgres Guide : http://postgresguide.com/

### Check postgres state in 60s:

Postgres Explain Beatify: https://explain.depesz.com/

```bash
# dont forget to replace your version
sudo du -csh -t 100M /var/lib/postgresql/10/main/* # check base and pg_xlog size
```

```sql
-- all databases and their sizes
select * from pg_user;

-- all tables and their size, with/without indexes
select datname, pg_size_pretty(pg_database_size(datname))
from pg_database
order by pg_database_size(datname) desc;

-- show queries that runs more than 2 minutes
SELECT now() - query_start as "runtime", usename, datname, waiting, state, query
  FROM  pg_stat_activity
  WHERE now() - query_start > '2 minutes'::interval
 ORDER BY runtime DESC;

-- show running queries
SELECT pid, age(clock_timestamp(), query_start), usename, query 
FROM pg_stat_activity 
WHERE query != '<IDLE>' AND query NOT ILIKE '%pg_stat_activity%' 
ORDER BY query_start desc;
```

Check pgBouncer state:

```sql
psql -p 6432 pgbouncer # Only users listed in the configuration parameters admin_users or stats_users are allowed to log in
SHOW STATS
SHOW TOTALS # is there any anomalies across stats?
```

## DB dump / backup

Official Postgres script for automated backups: http://wiki.postgresql.org/wiki/Automated_Backup_on_Linux

Easier way: https://www.depesz.com/2013/09/11/how-to-make-backups-of-postgresql/

Quick snippet for PG and MySQL backup:

```bash
pg_dump -Fc --schema='public' --exclude-table='excl.table' -h <host> -U <user> -W -v <db> > db.dump.$(date +%F) # dump a database into a custom-format archive
pg_restore -d newdb db.dump # reload an archive file into a (freshly created) database named newdb

mysqldump -u <user> -p <password> -h <host> --quick --single-transaction --databases db1 --result-file=db-backup-$(date +%F).sql 
# option --single-transaction sets the transaction isolation mode to REPEATABLE READ and sends a START TRANSACTION SQL statement to the server before dumping data 
mysql db1 < dump.sql
```

### Postgres deployment and configuration:

Postgres config generator: https://pgtune.leopard.in.ua/#/

Deployment templates:

A Template for PostgreSQL HA with ZooKeeper, etcd or Consul: https://github.com/zalando/patroni

Postgres + Pacemaker + Corosync: https://github.com/clusterlabs/PAF

## Development

### Helpers

Online generators:

Curl <-> Go https://mholt.github.io/curl-to-go/ https://mholt.github.io/json-to-go/

JSON <-> Any language https://quicktype.io

CURL -> Python/PHP/JSON/etc https://curl.trillworks.com

### Bash

How to write good bash: https://blog.yossarian.net/2020/01/23/Anybody-can-write-good-bash-with-a-little-effort

## Simple perfomance test

Simple example with [wrk](https://github.com/wg/wrk):

```bash
wrk -t12 -c400 -d30s http://127.0.0.1:8080/index.html
```

More advanced example with [hey](https://github.com/rakyll/hey) (written in Go). POST Query with cookies:

```bash
hey -c 200 -z 60s -m POST -T "application/json" -H "Cookie: cookie1=val1" -H "Content-Type: application/json"  -d '{"key1": "val1"}' http://localhost:8000/endpoint
```