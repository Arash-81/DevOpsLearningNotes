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
      has_schema_privilege('iran_map_read_only', n.nspname, 'USAGE') AS usage,
      has_schema_privilege('iran_map_read_only', n.nspname, 'CREATE') AS create
  FROM pg_namespace n
  WHERE n.nspname NOT IN ('information_schema', 'pg_catalog', 'pg_toast')
  ORDER BY n.nspname;
  ```
