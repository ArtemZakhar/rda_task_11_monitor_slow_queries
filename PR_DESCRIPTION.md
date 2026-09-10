## Summary

Compared read performance of `Products1` (InnoDB) vs `Products2` (MyISAM) using the MySQL slow query log.
`Products2` (MyISAM) was slower on average, so its `CREATE TABLE` and `INSERT` statements were removed from `task.sql`.
The final script keeps only `Products1` (InnoDB) with consolidated multi-row INSERT.

## Slow query log configuration

Added to `/etc/mysql/mysql.conf.d/mysqld.cnf` on the VM:

```ini
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time = 0
```

Restart and status check:

```bash
sudo systemctl restart mysql
sudo systemctl status mysql
```

## Database setup and queries (VM MySQL client)

```bash
mysql -u root -p < task.sql   # initial run with both Products1 and Products2
mysql -u root -p ShopDB
```

Inside the MySQL client, each query was executed **10 times**:

```sql
SELECT * FROM Products1 WHERE Name = "AwersomeProduct42";
SELECT * FROM Products2 WHERE Name = "AwersomeProduct42";
```

Terminal loop used for benchmarking:

```bash
for i in $(seq 1 10); do
  mysql -u root -p ShopDB -e 'SELECT * FROM Products1 WHERE Name = "AwersomeProduct42";'
done

for i in $(seq 1 10); do
  mysql -u root -p ShopDB -e 'SELECT * FROM Products2 WHERE Name = "AwersomeProduct42";'
done
```

## Slow log analysis

Filter commands:

```bash
grep -B1 'SELECT \* FROM Products1' /var/log/mysql/mysql-slow.log | grep Query_time
grep -B1 'SELECT \* FROM Products2' /var/log/mysql/mysql-slow.log | grep Query_time
```

Sample slow log excerpts:

```
# Query_time: 0.000147  Lock_time: 0.000002 Rows_sent: 1  Rows_examined: 60
SELECT * FROM Products1 WHERE Name = "AwersomeProduct42";

# Query_time: 0.000252  Lock_time: 0.000003 Rows_sent: 1  Rows_examined: 60
SELECT * FROM Products2 WHERE Name = "AwersomeProduct42";
```

| Table | Engine | Runs | Avg Query_time |
|-------|--------|------|----------------|
| Products1 | InnoDB | 10 | ~0.000092s |
| Products2 | MyISAM | 10 | ~0.000206s |

**Conclusion:** `Products2` (MyISAM) is ~2.2x slower on average → removed from `task.sql`.

## Final state

- `task.sql` contains only `Products1` (InnoDB) with a single multi-row INSERT.
- `Products2` (MyISAM) removed as the slower engine.
