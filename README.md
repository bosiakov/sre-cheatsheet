# Operations Engineer Cheatsheet

Collection of cheatsheets for performing a complete check of system health, database administration, performance benchmarks, and documentation links.

Target audience: DevOps, SRE, System Administrators, and everyone who is on duty.

## Linux Performance Checklist

USE Method: http://www.brendangregg.com/USEmethod/use-linux.html

### CPU

Linux kernel CPU Load guide: https://www.kernel.org/doc/html/latest/admin-guide/cpu-load.html

Linux Load Averages: http://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html

```bash
uptime # check load avg
vmstat 1 # system-wide utilization. Swapping?
mpstat -P ALL 1 # CPU balance
pidstat 1 # per-process CPU
```

### Memory

```bash
free -m # memory usage
```

### Disk

```bash
iostat -xnz 1 # any disk IO? no? good
df -h # are file systems nearly full?
du -hs /var/log/* | sort -rh | less # check log sizes
```

### Network

[List of TCP and UDP port numbers](https://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers)

Check connection statistics ([source](https://gist.github.com/mindfuckup/c82e4ddae16c8d68a67a9699ff7c4c20)):

```
curl -w "DNS: %{time_namelookup}\nTCP: %{time_connect}\nTLS: %{time_appconnect}\nTotal: %{time_total}\n" -o /dev/null -s https://example.net/test
```

Verify hostname resolution:

```
host example.com 1.1.1.1
nslookup example.com 1.1.1.1
dig @1.1.1.1 +short example.com
```

Verify TCP/UDP port accessibility:

```
telnet localhost 5672 # TCP
nc -v localhost 25 # TCP
nc -uv localhost 53 # UDP
```

Inspect open ports, TCP and UDP connections:

```bash
sudo netstat --all --numeric --tcp --programs
sudo lsof -n -i4TCP:5672 | grep LISTEN # display OS processes that listen on port 5672 and use IPv4
sudo ss --tcp -f inet --listening --numeric --processes # display listening TCP sockets that use IPv4 and their OS processes
lsof -ni TCP:3306 | awk '{print $2}' | uniq -c
# 3306 - default MYSQL port
# 5432 - default PSQL port
# 6432 - default pgBouncer port
```

Inspect network IO and TCP stats:

```bash
sar -n DEV 1 # network IO
sar -n TCP,ETCP 1 # TCP stats
```

## Storage

Database actual isolation levels: https://github.com/ept/hermitage

### Postgres

Postgres docs: https://www.postgresql.org/docs/current/index.html

Postgres Guide : http://postgresguide.com/

Postgres Observability: https://pgstats.dev/

#### Configuration

Server Configuration: https://www.postgresql.org/docs/current/runtime-config.html

Postgres Config generator: https://pgtune.leopard.in.ua/#/

```
pg_lsclusters # show information about all PostgreSQL clusters
```

In PSQL shell:

```
SHOW hba_file; -- Host Based Authentication Configuration
-- usually located in /etc/postgresql/xx/main/pg_hba.conf

SHOW config_file; -- PostgreSQL Server Configuration
-- usually located in /etc/postgresql/xx/main/postgresql.conf
```

Authentication Methods: https://www.postgresql.org/docs/current/auth-methods.html

#### Explain

Postgres Explain Beatify: https://explain.depesz.com/

Postgres Explain Visualizer: https://tatiyants.com/pev/#/plans/new

#### Disk usage

Detailed PostGres Disk Usage: https://wiki.postgresql.org/wiki/Disk_Usage

Check DB sizes:

```sql
\l+

-- size of one table 
\c dbname
SELECT pg_size_pretty( pg_total_relation_size('tablename'));
```

Check the number of connections:

```
SELECT * FROM pg_stat_activity;
```

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

MySQL reference: https://dev.mysql.com/doc/refman/8.0/en/

#### Configuration

MySQL Server Options: https://dev.mysql.com/doc/refman/8.0/en/server-options.html

Default options are read from the following files in the given order:

```
/etc/mysql/my.cnf ~/.my.cnf /usr/etc/my.cnf
```

```
# check system process list using 
ps ax | grep '[m]ysqld'

# Warning, you can easily overwrite pidfile for running instance 
# use --pid-file
mysqld --help --verbose --pid-file=XYZ | grep -A 1 "Default options"
```

```
mysql> SHOW VARIABLES;
```

#### Server status

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

Check Redis state:

```bash
redis-cli info # Complete info including network, CPU, memory, persistence and cluster settings
redis-cli monitor # Warning: Because MONITOR streams back all commands, its use comes at a cost

redis-cli client list # returns information and statistics about the client connections
redis-cli dbsize # the number of keys in the currently-selected database. New connections always use the database 0.
```

Redis latency problems checklist: https://redis.io/topics/latency

```bash
# latency is a measure of how long does a ping command take to receive a response from the server
redis-cli --latency -h `host` -p `port`
redis-cli --intrinsic-latency 100 # we can't ask better than intrinsic latency value 
```
Latency monitoring framework: https://redis.io/topics/latency-monitor

```bash
CONFIG SET latency-monitor-threshold 100 # latency threshold in milliseconds
LATENCY LATEST # the latest latency samples for all events
LATENCY DOCTOR # a human-readable latency analysis report
```

Redis slow queries log: https://redis.io/commands/slowlog

```bash
SLOWLOG GET 10
# Result:
# An identifier
# The unix timestamp
# execution time in microseconds
# arguments of the command

SLOWLOG LEN # the length of the slow log
SLOWLOG RESET # reset the slow log
```

Warning: Do not use **KEYS** in Production. It may ruin performance when it is executed against large databases. This command is intended for debugging. Read more: https://redis.io/commands/keys

## JVM

How different Java versions behave in a container: https://merikan.com/2019/04/jvm-in-a-container/

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


## Post Mortem

Reading postmortems: http://danluu.com/postmortem-lessons/

A List of Post-mortems: https://github.com/danluu/post-mortems

## Simple perfomance test

Simple example with [wrk](https://github.com/wg/wrk):

```bash
wrk -t12 -c400 -d30s http://127.0.0.1:8080/index.html
```

Apache Bench [ab](https://httpd.apache.org/docs/2.4/programs/ab.html):

```bash
ab -r -k -n 100000 -c 100 http://localhost:9000/
```

More advanced example with [hey](https://github.com/rakyll/hey) (written in Go). POST Query with cookies:

```bash
hey -c 200 -z 60s -m POST -T "application/json" -H "Cookie: cookie1=val1" -H "Content-Type: application/json"  -d '{"key1": "val1"}' http://localhost:8000/endpoint
```

Redis benchmark (https://redis.io/topics/benchmarks):

```
redis-benchmark -c 10 -n 100000 -q
```

Postgres benchmark (https://www.postgresql.org/docs/current/pgbench.html)

```
pgbench -i -c 10 -j 10 -t 10000 -h host -p port -U user dbname
```