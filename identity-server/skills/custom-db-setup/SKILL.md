---
name: custom-db-setup
description: Use when setting up a custom database for WSO2 Identity Server. Triggers on requests like "set up database for IS", "configure MySQL for Identity Server", "connect postgres to WSO2 IS", "set up IS with MSSQL", "connect DB2 to WSO2 IS", "connect Oracle to WSO2 IS", or any request to wire a database to WSO2 Identity Server.
tools: Read, Write, Edit, Bash, Glob, Grep
---

# WSO2 Identity Server — Database Setup

You are setting up a Docker-based database and wiring it to WSO2 Identity Server (IS). Work through each step in order. If any command fails unexpectedly, show the user the error output and ask how to proceed before continuing.

## DB Config Reference

Use this table throughout all steps. Do not re-ask for values the user already provided.

| DB Type    | Docker Image                              | Port  | Default User      | Default Password | Driver prefix    |
|------------|-------------------------------------------|-------|-------------------|------------------|------------------|
| MySQL      | `mysql:<version>`                         | 3306  | `root`            | `root`           | `mysql-connector`|
| PostgreSQL | `postgres:<version>`                      | 5432  | `postgres`        | `root`           | `postgresql`     |
| MSSQL      | `mcr.microsoft.com/azure-sql-edge:latest` | 1433  | `sa`              | `Root@1234`      | `mssql-jdbc`     |
| DB2        | `ibmcom/db2-amd64`                        | 50000 | `db2inst1`        | `wso2carbon`     | `db2jcc`         |
| Oracle     | `gvenzl/oracle-free`                      | 1521  | `wso2carbon`      | `wso2carbon`     | `ojdbc`          |

**Container naming convention:** `wso2is_<db>_<version>` — e.g. `wso2is_mysql_8.0`, `wso2is_postgres_17`, `wso2is_mssql_azure-sql-edge`, `wso2is_db2_amd`, `wso2is_oracle_free`.

**JDBC URL templates:**

| DB Type    | JDBC URL                                                                                        |
|------------|-------------------------------------------------------------------------------------------------|
| MySQL      | *(hostname/port style — no URL field needed)*                                                   |
| PostgreSQL | *(hostname/port style — no URL field needed)*                                                   |
| MSSQL      | `jdbc:sqlserver://127.0.0.1:1433;databaseName=<db_name>;SendStringParametersAsUnicode=false;trustServerCertificate=true` |
| DB2        | `jdbc:db2://localhost:50000/<db_name>:currentSchema=DB2INST1;`                                  |
| Oracle     | `jdbc:oracle:thin:@//localhost:1521/FREEPDB1`                                                   |

---

## Step 1 — Gather required inputs

Ask the user for anything not yet provided:

- **IS location** — path to the IS zip or extracted IS home directory
- **Database type** — `mysql`, `mssql`, `postgres`, `db2`, or `oracle`
- **Database version** — e.g. `8.0` / `9.5` (MySQL), `16` / `17` (PostgreSQL), `azure-sql-edge` (MSSQL on ARM), `db2-amd64` (DB2), `free` (Oracle 23c)
- **Database name** — default `regdb`
- **DB username / password** — use defaults from the reference table if not provided

> **Oracle on Mac:** Always use `gvenzl/oracle-free` (Oracle 23c Free, multi-arch). `gvenzl/oracle-xe` (21c) is AMD64-only and fails on Apple Silicon with `ORA-00443: PMON did not start`. If using Colima, start with `colima start --network-address` so ports are reachable via `localhost`.

> **DB2 on Apple Silicon:** `ibmcom/db2-amd64` is x86_64 and runs under Rosetta via `--platform=linux/amd64`. The image is ~3.5 GB — verify disk space before pulling (see Step 4).

## Step 2 — Resolve IS home directory

Set `IS_HOME` to the extracted directory. Confirm `IS_HOME` contains `repository/conf/deployment.toml` — if not, stop and ask the user to verify the path.

## Step 3 — Check Docker is running

```bash
docker info > /dev/null 2>&1 && echo "Docker OK" || echo "Docker not running"
```

If Docker is not running: tell the user to start Docker Desktop and **wait for them to confirm** before continuing. Re-run the check once they do.

## Step 4 — Start the database container

**First, check if a container with the target name already exists:**

```bash
docker ps -a --filter "name=<container_name>" --format "{{.Status}}"
```

- If `Up`: reuse it, skip creation, continue to Step 5.
- If `Exited`: `docker start <container_name>`, wait for readiness, continue to Step 5.
- If not found: create it using the commands below.

---

### MySQL

**Before creating — verify disk space for DB2 only.** (Skip this for MySQL/PostgreSQL/MSSQL/Oracle.)

```bash
docker run --name wso2is_mysql_<version> \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=<password> \
  -e MYSQL_DATABASE=<db_name> \
  -d mysql:<version>
```

Wait for readiness (up to 30s):

```bash
for i in $(seq 1 30); do
  docker exec wso2is_mysql_<version> mysqladmin ping -u root -p<password> --silent 2>/dev/null && echo "MySQL ready" && break
  echo "Waiting for MySQL... ($i/30)"; sleep 1
done
```

### PostgreSQL

```bash
docker run --name wso2is_postgres_<version> \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=<password> \
  -e POSTGRES_DB=<db_name> \
  -d postgres:<version>
```

Wait for readiness (up to 30s):

```bash
for i in $(seq 1 30); do
  docker exec wso2is_postgres_<version> pg_isready -U postgres 2>/dev/null && echo "PostgreSQL ready" && break
  echo "Waiting for PostgreSQL... ($i/30)"; sleep 1
done
```

### MSSQL (Azure SQL Edge)

> ARM-compatible. Uses `MSSQL_SA_PASSWORD` (not `SA_PASSWORD`). Does **not** include `sqlcmd` — use host `sqlcmd` (`brew install sqlcmd`) connecting to `localhost,1433`. SA password must be min 8 chars with upper, lower, digit, and special character.

```bash
docker run --name wso2is_mssql_azure-sql-edge \
  -p 1433:1433 \
  -e MSSQL_SA_PASSWORD=<password> \
  -e ACCEPT_EULA=Y \
  -d mcr.microsoft.com/azure-sql-edge:latest
```

Wait for readiness (up to 60s — takes longer than other DBs):

```bash
for i in $(seq 1 60); do
  sqlcmd -S localhost,1433 -U sa -P "<password>" -C \
    -Q "SELECT 1" > /dev/null 2>&1 && echo "MSSQL ready" && break
  echo "Waiting for MSSQL... ($i/60)"; sleep 1
done
```

### DB2

> **Before pulling the image (~3.5 GB), verify VM disk space:**

```bash
docker run --rm alpine df -h /
```

You need at least **8 GB free**. If not:
- **Docker Desktop**: Settings → Resources → Advanced → Virtual disk limit → increase and Apply & Restart
- **Rancher Desktop**: Preferences → Virtual Machine → Hardware → Disk size → increase and Apply

Then prune unused layers: `docker system prune -f`

```bash
docker run --platform=linux/amd64 -itd --name wso2is_db2_amd \
  --privileged=true \
  -p 50000:50000 \
  -e LICENSE=accept \
  -e DB2INST1_PASSWORD=<password> \
  -e DB2INSTANCE=db2inst1 \
  -e DBNAME=<db_name> \
  ibmcom/db2-amd64
```

Wait for readiness (up to 6 minutes — polls every 3s):

```bash
for i in $(seq 1 120); do
  docker exec wso2is_db2_amd su - db2inst1 -c "db2 connect to <db_name>" \
    > /dev/null 2>&1 && echo "DB2 ready" && break
  echo "Waiting for DB2... ($i/120)"; sleep 3
done
```

### Oracle 23c Free

> Uses `FREEPDB1` as the PDB service name. `ORACLE_PASSWORD` sets the password for `SYS`/`SYSTEM`. A dedicated WSO2 user is created in Step 5.

```bash
docker run -d \
  --name wso2is_oracle_free \
  -p 1521:1521 \
  -e ORACLE_PASSWORD=<oracle_sys_password> \
  -v oracle-data:/opt/oracle/oradata \
  gvenzl/oracle-free
```

Wait for readiness (up to 5 minutes — polls every 5s):

```bash
for i in $(seq 1 60); do
  docker logs wso2is_oracle_free 2>/dev/null | grep -q "DATABASE IS READY TO USE" && echo "Oracle ready" && break
  echo "Waiting for Oracle... ($i/60)"; sleep 5
done
```

> If `localhost` is unreachable (Colima without `--network-address`): `colima list` → use the `ADDRESS` as the host in all subsequent commands and in `deployment.toml`.

## Step 5 — Create the database / schema

MySQL, PostgreSQL, and DB2 auto-create the database from the env vars in Step 4 (`MYSQL_DATABASE`, `POSTGRES_DB`, `DBNAME`). Skip to Step 6 for those.

**MSSQL** — create the database manually:

```bash
sqlcmd -S localhost,1433 -U sa -P "<password>" -C \
  -Q "IF NOT EXISTS (SELECT name FROM sys.databases WHERE name = '<db_name>') CREATE DATABASE [<db_name>]"
```

**Oracle** — create a dedicated WSO2 user/schema in `FREEPDB1`. The username becomes the schema name.

> If `<oracle_sys_password>` contains `@`, quote it in the connection string: `'sys/"<password>"@localhost/FREEPDB1 as sysdba'` — unquoted `@` is parsed as the host delimiter.

```bash
docker exec wso2is_oracle_free bash -c "
echo 'CREATE USER <username> IDENTIFIED BY \"<password>\";
GRANT CREATE SESSION TO <username>;
GRANT DBA TO <username>;
ALTER USER <username> QUOTA UNLIMITED ON USERS;
EXIT;' \
| sqlplus -s 'sys/\"<oracle_sys_password>\"@localhost/FREEPDB1 as sysdba'"
```

## Step 6 — Install the JDBC driver

Check if a compatible driver already exists in `<IS_HOME>/repository/components/lib/`:

```bash
ls "<IS_HOME>/repository/components/lib/" | grep -i "<driver-prefix>"
```

Use the driver prefix from the reference table. If a driver is already present, inform the user and skip download.

If not present:

**MySQL** — via Maven if available, otherwise manual download from `https://dev.mysql.com/downloads/connector/j/`:
```bash
mvn dependency:get -Dartifact=com.mysql:mysql-connector-j:8.0.33:jar -Ddest="<IS_HOME>/repository/components/lib/"
```

**PostgreSQL** — direct download:
```bash
curl -L "https://jdbc.postgresql.org/download/postgresql-42.7.3.jar" \
  -o "<IS_HOME>/repository/components/lib/postgresql-42.7.3.jar"
```

**MSSQL** — download from Microsoft, extract, copy `mssql-jdbc-<version>.jre11.jar` to `<IS_HOME>/repository/components/lib/`.
Download page: `https://learn.microsoft.com/en-us/sql/connect/jdbc/download-microsoft-jdbc-driver-for-sql-server`

**DB2** — copy directly from the running container (bundled in the image):
```bash
docker cp wso2is_db2_amd:/opt/ibm/db2/V11.5/java/db2jcc4.jar \
  "<IS_HOME>/repository/components/lib/db2jcc4.jar"
```

**Oracle** — via Maven (available on Maven Central since 19c, no Oracle account needed):
```bash
mvn dependency:get \
  -Dartifact=com.oracle.database.jdbc:ojdbc11:23.4.0.24.05:jar \
  -Ddest="<IS_HOME>/repository/components/lib/"
```
Or direct download:
```bash
curl -L "https://repo1.maven.org/maven2/com/oracle/database/jdbc/ojdbc11/23.4.0.24.05/ojdbc11-23.4.0.24.05.jar" \
  -o "<IS_HOME>/repository/components/lib/ojdbc11-23.4.0.24.05.jar"
```

Confirm the file is in place after any download:
```bash
ls "<IS_HOME>/repository/components/lib/" | grep -i "<driver-prefix>"
```

## Step 7 — Check if database is already initialized

Before running init scripts, check whether the database already has tables.

### MySQL
```bash
TABLE_COUNT=$(docker exec wso2is_mysql_<version> mysql -u root -p<password> -sNe \
  "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='<db_name>';" 2>/dev/null)
echo "Tables found: $TABLE_COUNT"
```

### PostgreSQL
```bash
TABLE_COUNT=$(docker exec wso2is_postgres_<version> psql -U postgres -d <db_name> -tAc \
  "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='public';" 2>/dev/null)
echo "Tables found: $TABLE_COUNT"
```

### MSSQL (Azure SQL Edge)
```bash
TABLE_COUNT=$(sqlcmd -S localhost,1433 -U sa -P "<password>" -d <db_name> -C -h -1 \
  -Q "SET NOCOUNT ON; SELECT COUNT(*) FROM sys.tables" 2>/dev/null | tr -d ' ')
echo "Tables found: $TABLE_COUNT"
```

### DB2

> `su - db2inst1 -c "..."` loses DB2 connection context between semicolons. Use the `bash -c` wrapper to keep the session alive across both statements.

```bash
TABLE_COUNT=$(docker exec wso2is_db2_amd bash -c \
  "su - db2inst1 -c 'db2 connect to <db_name>; db2 SELECT COUNT\(*\) FROM syscat.tables WHERE tabschema = UPPER\(CURRENT_USER\)'" \
  2>/dev/null | grep -E "^[[:space:]]*[0-9]" | tr -d ' ')
echo "Tables found: $TABLE_COUNT"
```

### Oracle

> Use `echo` with double-quoted SQL inside the `bash -c` block — `printf` with nested single-quote escaping breaks the string literal inside `UPPER('...')`.

```bash
TABLE_COUNT=$(docker exec wso2is_oracle_free bash -c "
echo \"SET HEADING OFF
SET FEEDBACK OFF
SET PAGESIZE 0
SELECT COUNT(*) FROM all_tables WHERE OWNER = UPPER('<username>');
EXIT;\" | sqlplus -s '<username>/<password>@localhost/FREEPDB1'" 2>/dev/null | grep -E "^[[:space:]]*[0-9]" | tr -d ' ')
echo "Tables found: $TABLE_COUNT"
```

---

**If `TABLE_COUNT > 0`**, ask the user:

> "The database `<db_name>` already contains $TABLE_COUNT table(s). How would you like to proceed?
> 1. **Drop and recreate** — delete all existing data and start fresh
> 2. **Use a different name** — enter a new database name"

Wait for the user's choice before continuing.

**Option 1 — Drop and recreate:**

MySQL:
```bash
docker exec wso2is_mysql_<version> mysql -u root -p<password> \
  -e "DROP DATABASE \`<db_name>\`; CREATE DATABASE \`<db_name>\`;"
```

PostgreSQL:
```bash
docker exec wso2is_postgres_<version> psql -U postgres \
  -c "DROP DATABASE \"<db_name>\"; CREATE DATABASE \"<db_name>\";"
```

MSSQL:
```bash
sqlcmd -S localhost,1433 -U sa -P "<password>" -C \
  -Q "ALTER DATABASE [<db_name>] SET SINGLE_USER WITH ROLLBACK IMMEDIATE; DROP DATABASE [<db_name>]; CREATE DATABASE [<db_name>];"
```

DB2:
```bash
docker exec wso2is_db2_amd su - db2inst1 -c \
  "db2 connect to <db_name> && db2 drop database <db_name> && db2 create database <db_name> using codeset UTF-8 territory us"
```

Oracle (drop and recreate user/schema in `FREEPDB1`):
```bash
docker exec wso2is_oracle_free bash -c "
echo 'DROP USER <username> CASCADE;
CREATE USER <username> IDENTIFIED BY \"<password>\";
GRANT CREATE SESSION TO <username>;
GRANT DBA TO <username>;
ALTER USER <username> QUOTA UNLIMITED ON USERS;
EXIT;' \
| sqlplus -s 'sys/\"<oracle_sys_password>\"@localhost/FREEPDB1 as sysdba'"
```

**Option 2 — Use a different name:** take the new name from the user and create a fresh database using the appropriate command from Step 5. Use the new name as `<db_name>` for all subsequent steps.

## Step 8 — Run the database initialization scripts

Scripts are in `<IS_HOME>/dbscripts/`. Run both the shared schema script and the identity schema script.

### MySQL
```bash
docker exec -i wso2is_mysql_<version> mysql -u root -p<password> <db_name> \
  < "<IS_HOME>/dbscripts/mysql.sql"

docker exec -i wso2is_mysql_<version> mysql -u root -p<password> <db_name> \
  < "<IS_HOME>/dbscripts/identity/mysql.sql"
```

### PostgreSQL
```bash
docker exec -i wso2is_postgres_<version> psql -U postgres -d <db_name> \
  < "<IS_HOME>/dbscripts/postgresql.sql"

docker exec -i wso2is_postgres_<version> psql -U postgres -d <db_name> \
  < "<IS_HOME>/dbscripts/identity/postgresql.sql"
```

### MSSQL (Azure SQL Edge)
```bash
sqlcmd -S localhost,1433 -U sa -P "<password>" -d <db_name> -C \
  -i "<IS_HOME>/dbscripts/mssql.sql"

sqlcmd -S localhost,1433 -U sa -P "<password>" -d <db_name> -C \
  -i "<IS_HOME>/dbscripts/identity/mssql.sql"
```

### DB2

> **Two critical gotchas:**
> 1. **Use `-td/ -vf`, not `-tvf`** — WSO2 DB2 scripts use `/` as the statement delimiter. `-tvf` defaults to `;` and treats the entire file as one statement, failing with `SQL0104N`.
> 2. **Connection context is lost between `&&`-chained commands** — write a shell script into the container and run it via `su - db2inst1 -c "bash /tmp/script.sh"` to keep the session alive.

```bash
docker cp "<IS_HOME>/dbscripts/db2.sql" wso2is_db2_amd:/tmp/db2.sql
docker cp "<IS_HOME>/dbscripts/identity/db2.sql" wso2is_db2_amd:/tmp/db2_identity.sql

docker exec wso2is_db2_amd bash -c "cat > /tmp/run_db2.sh << 'SCRIPT'
#!/bin/bash
db2 connect to <db_name>
db2 -td/ -vf /tmp/db2.sql
db2 connect reset
SCRIPT
chmod +x /tmp/run_db2.sh"

docker exec wso2is_db2_amd su - db2inst1 -c "bash /tmp/run_db2.sh" 2>&1 | tail -20

docker exec wso2is_db2_amd bash -c "cat > /tmp/run_db2_identity.sh << 'SCRIPT'
#!/bin/bash
db2 connect to <db_name>
db2 -td/ -vf /tmp/db2_identity.sql
db2 connect reset
SCRIPT
chmod +x /tmp/run_db2_identity.sh"

docker exec wso2is_db2_amd su - db2inst1 -c "bash /tmp/run_db2_identity.sh" 2>&1 | tail -20
```

> `dbscripts/db2.sql` contains the full shared schema (Carbon registry, UM_*, org hierarchy, etc.) — not just the registry. `dbscripts/identity/db2.sql` is the identity schema (IDN_*, OAuth, SAML, etc.). Both are required; together they create ~243 tables on a fresh IS 7.2.x install.

### Oracle

> `WHENEVER SQLERROR CONTINUE` lets the script skip "object already exists" errors (ORA-00955, ORA-02260) safely on re-runs.

```bash
docker cp "<IS_HOME>/dbscripts/oracle.sql" wso2is_oracle_free:/tmp/oracle.sql
docker cp "<IS_HOME>/dbscripts/identity/oracle.sql" wso2is_oracle_free:/tmp/oracle_identity.sql

docker exec wso2is_oracle_free bash -c "
echo 'WHENEVER SQLERROR CONTINUE
@/tmp/oracle.sql
EXIT;' | sqlplus -s '<username>/<password>@localhost/FREEPDB1'" 2>&1 | tail -30

docker exec wso2is_oracle_free bash -c "
echo 'WHENEVER SQLERROR CONTINUE
@/tmp/oracle_identity.sql
EXIT;' | sqlplus -s '<username>/<password>@localhost/FREEPDB1'" 2>&1 | tail -30
```

If scripts fail with errors other than "object already exists", show them to the user.

## Step 9 — Update deployment.toml

Read `<IS_HOME>/repository/conf/deployment.toml`. Locate `[database.identity_db]` and `[database.shared_db]`. Replace existing blocks entirely (no duplicates) or append if they don't exist.

### MySQL
```toml
[database.identity_db]
type = "mysql"
hostname = "localhost"
name = "<db_name>"
port = "3306"
username = "<username>"
password = "<password>"

[database.shared_db]
type = "mysql"
hostname = "localhost"
name = "<db_name>"
port = "3306"
username = "<username>"
password = "<password>"
```

### PostgreSQL

> Note: the type string is `"postgre"` — not `"postgresql"`. This is correct per WSO2 documentation.

```toml
[database.identity_db]
type = "postgre"
hostname = "localhost"
name = "<db_name>"
port = "5432"
username = "<username>"
password = "<password>"

[database.shared_db]
type = "postgre"
hostname = "localhost"
name = "<db_name>"
port = "5432"
username = "<username>"
password = "<password>"
```

### MSSQL
```toml
[database.identity_db]
type = "mssql"
url = "jdbc:sqlserver://127.0.0.1:1433;databaseName=<db_name>;SendStringParametersAsUnicode=false;trustServerCertificate=true"
username = "<username>"
password = "<password>"

[database.shared_db]
type = "mssql"
url = "jdbc:sqlserver://127.0.0.1:1433;databaseName=<db_name>;SendStringParametersAsUnicode=false;trustServerCertificate=true"
username = "<username>"
password = "<password>"
```

### DB2
```toml
[database.identity_db]
type = "db2"
url = "jdbc:db2://localhost:50000/<db_name>:currentSchema=DB2INST1;"
username = "db2inst1"
password = "<password>"

[database.shared_db]
type = "db2"
url = "jdbc:db2://localhost:50000/<db_name>:currentSchema=DB2INST1;"
username = "db2inst1"
password = "<password>"
```

### Oracle

> The `<username>` is also the Oracle schema name. If `localhost` is unreachable (Colima), replace it with the Colima VM IP from `colima list`.

```toml
[database.identity_db]
type = "oracle"
url = "jdbc:oracle:thin:@//localhost:1521/FREEPDB1"
username = "<username>"
password = "<password>"

[database.shared_db]
type = "oracle"
url = "jdbc:oracle:thin:@//localhost:1521/FREEPDB1"
username = "<username>"
password = "<password>"
```

After editing, read the relevant section back and show the user the final config.

## Step 10 — Summary and start IS

Print a clear summary:

```
✅ Database setup complete

  Docker container : <container_name>
  DB type          : <db_type>
  DB name          : <db_name>  (<N> tables created)
  Host:Port        : localhost:<port>
  Username         : <username>

  deployment.toml  : <IS_HOME>/repository/conf/deployment.toml  ← updated
  JDBC driver      : <IS_HOME>/repository/components/lib/<driver.jar>
```

If any step failed, state which step and what went wrong. Do not start IS if any step failed.

Start WSO2 IS in the background and watch for startup or fatal errors:

```bash
sh "<IS_HOME>/bin/wso2server.sh" > /tmp/wso2is_startup.log 2>&1 &
for i in $(seq 1 12); do
  sleep 10
  if grep -q "WSO2 Carbon started in" /tmp/wso2is_startup.log 2>/dev/null; then
    echo "WSO2 IS started successfully"
    grep "WSO2 Carbon started in" /tmp/wso2is_startup.log
    break
  fi
  if grep -q "FATAL" /tmp/wso2is_startup.log 2>/dev/null; then
    echo "IS startup error detected:"
    grep -i "fatal" /tmp/wso2is_startup.log | tail -20
    break
  fi
  echo "Waiting for IS to start... ($i/12)"
done
```

If IS has not started within 120 seconds, tail the IS log for more detail:

```bash
tail -50 "<IS_HOME>/repository/logs/wso2carbon.log"
```

Report any ERROR or FATAL lines to the user. Continue polling `/tmp/wso2is_startup.log` until IS is up or a fatal error is confirmed.

---

## Notes

- `[database.shared_db]` and `[database.identity_db]` can point to the same database — standard for dev/test. Use separate databases for production.
- DB scripts are idempotent for table-creation errors. "Object already exists" errors from re-runs are safe to ignore.
- DB2 runs all commands as the `db2inst1` OS user inside the container — always via `su - db2inst1 -c "..."` or the shell script pattern.
- Oracle schemas are tied to users: there are no separate "databases" within the PDB. `<username>` is both the login and the schema that holds all WSO2 tables. Always target `FREEPDB1`, never `FREE` (the CDB root).
