## Postgres useful commands
Some interesting flags (to see all, use `-h` or `--help` depending on your psql version):
- `-E`: will describe the underlaying queries of the `\` commands (cool for learning!)
- `-l`: psql will list all databases and then exit (useful if the user you connect with doesn't has a default database, like at AWS RDS)

Most `\d` commands support additional param of `__schema__.name__` and accept wildcards like `*.*`

- `\?`: Show help (list of available commands with an explanation)
- `\q`: Quit/Exit
- `\c __database__`: Connect to a database
- `\d __table__`: Show table definition (columns, etc.) including triggers
- `\d+ __table__`: More detailed table definition including description and physical disk size
- `\l`: List databases
- `\dy`: List events
- `\df`: List functions
- `\di`: List indexes
- `\dn`: List schemas
- `\dt *.*`: List tables from all schemas (if `*.*` is omitted will only show SEARCH_PATH ones)
- `\dT+`: List all data types
- `\dv`: List views
- `\dx`: List all extensions installed
- `\df+ __function__` : Show function SQL code. 
- `\x`: Pretty-format query results instead of the not-so-useful ASCII tables
- `\copy (SELECT * FROM __table_name__) TO 'file_path_and_name.csv' WITH CSV`: Export a table as CSV
- `\des+`: List all foreign servers
- `\dE[S+]`: List all foreign tables
- `\! __bash_command__`: execute `__bash_command__` (e.g. `\! ls`)

User Related:
- `\du`: List users
- `\du __username__`: List a username if present.
- `create role __test1__`: Create a role with an existing username.
- `create role __test2__ noinherit login password __passsword__;`: Create a role with username and password.
- `set role __test__;`: Change role for current session to `__test__`.
- `grant __test2__ to __test1__;`: Allow `__test1__` to set its role as `__test2__`.
- `\deu+`: List all user mapping on server

## Postgres useul queries

- `SELECT * FROM pg_extension;`: Check Extensions enabled in postgres
- `SELECT * FROM pg_available_extension_versions;`: Show available extensions
- `SELECT * FROM pg_indexes WHERE tablename='__table_name__' AND schemaname='__schema_name__';`: Show table indexes
- `select * from pg_stat_user_tables;`: Check tuples of a table
- `select pg_size_pretty(pg_relation_size('__table_name__'));`: Check table size
- Check Permissions for a Specific User: 
 
  ```sql
  SELECT
      grantee,
      table_schema,
      table_name,
      privilege_type
  FROM information_schema.role_table_grants
  WHERE grantee = '__username__'
  ORDER BY table_name, privilege_type;
  ```
- Check Permissions on a Specific Table:

  ```sql
  SELECT
      grantee,
      privilege_type,
      is_grantable
  FROM information_schema.role_table_grants
  WHERE table_name = '__table_name__';
  ```
- Check Specific User privileges on tables:

  ```sql
  SELECT 
      table_schema,
      table_name,
      STRING_AGG(privilege_type, ', ' ORDER BY privilege_type) AS privileges
  FROM information_schema.role_table_grants
  WHERE grantee = '__username__'
  GROUP BY table_schema, table_name
  ORDER BY table_schema, table_name;
  ```
- Check Specific User privileges on schemas:
  
  ```sql
  SELECT 
      n.nspname AS schema,
      has_schema_privilege('__username__', n.nspname, 'USAGE') AS usage,
      has_schema_privilege('__username__', n.nspname, 'CREATE') AS create
  FROM pg_namespace n
  WHERE n.nspname NOT IN ('information_schema', 'pg_catalog', 'pg_toast')
  ORDER BY n.nspname;
  ```
- Get list of sessions of specific user
  ```sql
  SELECT
      pid,
      usename,
      application_name,
      client_addr,
      state,
      query_start,
      state_change
  FROM pg_stat_activity
  WHERE usename = '__username__'
    AND state = 'idle';
  ```

- Kill idle session of specific user
  ```sql
  SELECT pg_terminate_backend(pid)
  FROM pg_stat_activity
  WHERE usename = '__username__' 
    AND state = 'idle'
    AND pid <> pg_backend_pid();
  ```

## Performance & Statistics Queries

- Full query execution plan with runtime stats, buffer usage, and cost estimates — prefix any query with this to diagnose slow queries:

  ```sql
  EXPLAIN (ANALYZE, VERBOSE, COSTS, TIMING, BUFFERS)
  SELECT ...;
  ```



- Top 10 queries by total execution time (requires `pg_stat_statements` extension):

  ```sql
  SELECT round((100 * total_time / sum(total_time)
             OVER ())::numeric, 2) percent,
             round(total_time::numeric, 2) AS total,
             calls,
             round(mean_time::numeric, 2) AS mean,
             substring(query, 1, 200)
   FROM  pg_stat_statements
             ORDER BY total_time DESC
             LIMIT 10;
  ```

- Tables with the most sequential scans — useful for finding candidates for new indexes (ordered by total tuples read via seq scan):

  ```sql
  SELECT schemaname, relname, seq_scan, seq_tup_read,
         seq_tup_read / seq_scan AS avg, idx_scan
  FROM   pg_stat_user_tables
  WHERE  seq_scan > 0
  ORDER BY seq_tup_read DESC
  LIMIT  25;
  ```

- Index sizes and scan counts — shows how often each index is used and how much space it occupies:

  ```sql
  SELECT schemaname, relname, indexrelname, idx_scan,
         pg_size_pretty(pg_relation_size(indexrelid)) AS idx_size
  FROM   pg_stat_user_indexes;
  ```

- Average tuples returned per index scan — high values may indicate low-selectivity indexes:

  ```sql
  SELECT indexrelname,
         cast(idx_tup_read AS numeric) / idx_scan AS avg_tuples,
         idx_scan, idx_tup_read
  FROM pg_stat_user_indexes
         WHERE idx_scan > 0;
  ```

- Insert/update/delete ratio per table — useful for understanding table workload characteristics:

  ```sql
  SELECT relname,
         cast(n_tup_ins AS numeric) / (n_tup_ins + n_tup_upd + n_tup_del) AS ins_pct,
         cast(n_tup_upd AS numeric) / (n_tup_ins + n_tup_upd + n_tup_del) AS upd_pct,
         cast(n_tup_del AS numeric) / (n_tup_ins + n_tup_upd + n_tup_del) AS del_pct
  FROM pg_stat_user_tables
         ORDER BY relname;
  ```

- HOT update percentage per table — HOT (Heap Only Tuple) updates avoid index churn; low `hot_pct` means updates are causing many index writes:

  ```sql
  SELECT relname, n_tup_upd, n_tup_hot_upd,
         cast(n_tup_hot_upd AS numeric) / n_tup_upd AS hot_pct
  FROM pg_stat_user_tables
         WHERE n_tup_upd > 0 ORDER BY hot_pct;
  ```

- Index scan vs sequential scan ratio per table — low `idx_scan_pct` means the table is mostly accessed via seq scans, suggesting a missing index:

  ```sql
  SELECT schemaname, relname, seq_scan, idx_scan,
         cast(idx_scan AS numeric) / (idx_scan + seq_scan)
         AS idx_scan_pct
  FROM pg_stat_user_tables
         WHERE (idx_scan + seq_scan) > 0 ORDER BY idx_scan_pct;
  ```

- Index vs sequential tuple fetch ratio per table — low `idx_tup_pct` means most rows are fetched via seq scans rather than indexes:

  ```sql
  SELECT relname, seq_tup_read, idx_tup_fetch,
         cast(idx_tup_fetch AS numeric) / (idx_tup_fetch + seq_tup_read)
         AS idx_tup_pct
  FROM pg_stat_user_tables
         WHERE (idx_tup_fetch + seq_tup_read) > 0 ORDER BY idx_tup_pct;
  ```
