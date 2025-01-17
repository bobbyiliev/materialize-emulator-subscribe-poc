# Materialize Kafka Data Generation Demo

This demo sets up a pipeline using Materialize, Redpanda (Kafka), and the Materialize datagen tool to generate and process streaming data.

## Setup

1. Clone the repository:
```bash
git clone git@github.com:bobbyiliev/materialize-emulator-subscribe-poc.git
cd materialize-emulator-subscribe-poc
```

2. Start the services:
```bash
docker compose up -d
```

3. Wait for all services to be healthy (usually takes about 30-45 seconds)
```bash
docker compose ps
```

## Connect to Materialize

Connect to the Materialize instance:
```bash
psql -h localhost -p 6877 -U mz_system materialize
```

## Create Required Objects

1. **Create a Secret for PostgreSQL Password:**
    ```sql
    CREATE SECRET pgpass AS 'postgres';
    ```

1. **Create PostgreSQL Connection:**
    ```sql
    CREATE CONNECTION pg_connection TO POSTGRES (
        HOST 'postgres',
        PORT 5432,
        USER 'postgres',
        PASSWORD SECRET pgpass,
        SSL MODE 'disable',
        DATABASE 'postgres'
    );
    ```

1. **Create the Source from PostgreSQL:**
    ```sql
    CREATE SOURCE mz_source
    FROM POSTGRES CONNECTION pg_connection (PUBLICATION 'mz_source')
    FOR ALL TABLES;
    ```

1. **Create a Materialized View:**
    ```sql
    CREATE MATERIALIZED VIEW datagen_view AS
        SELECT * FROM products;
    ```

1. **Verify Data Flow:**
    ```sql
    SELECT * FROM datagen_view LIMIT 5;
    ```

> [!TIP]
> `CREATE INDEX` creates an in-memory index on a source, view, or materialized view. For more information, see the [Materialize documentation](https://materialize.com/docs/sql/create-index/).
> In Materialize, indexes store query results in memory within a [cluster](https://materialize.com/docs/concepts/clusters/), and keep these results incrementally updated as new data arrives. By making up-to-date results available in memory, indexes can help [optimize query performance](https://materialize.com/docs/transform-data/optimization/), both when serving results and maintaining resource-heavy operations like joins.

4. Create an index to optimize queries (optional):
```sql
CREATE INDEX datagen_view_idx ON datagen_view (id);
```

> [!TIP]
> Running `SUBSCRIBE` on an unindexed view can be slow and resource-intensive as it requires a full scan of the view/table/source.

## Verify Setup

Check that data is flowing:
```sql
SELECT * FROM datagen_view LIMIT 5;
```

Check the number of records:
```sql
SELECT count(*) FROM datagen_view;
```

## Configuration Details

- The datagen service will generate:
  - 10,024 records (`-n 10024`)
  - With a write interval of 2000ms (`-w 2000`)
  - In Postgres format (`-f postgres`)
  - Using the schema defined in `/schemas/products.sql`

## Troubleshooting

If you don't see data flowing:
1. Check service health:
```bash
docker compose ps
```

2. Check Postgres logs:
```bash
docker compose logs postgres
```

3. Check datagen logs:
```bash
docker compose logs datagen
```

4. Verify the source:
```sql
SHOW SOURCES;
```

5. Check the source status:
```sql
SELECT * FROM mz_internal.mz_source_statuses;
```
   If the source status is not with `running` status, the `SUBSCRIBE` command will not return any data as no new data is received.

## Using `SUBSCRIBE`

You can use the `SUBSCRIBE` command to see the data as it flows in:
```sql
COPY (SUBSCRIBE datagen_view) TO STDOUT;
-- Subscribe with no snapshot:
COPY (SUBSCRIBE datagen_view WITH(SNAPSHOT FALSE)) TO STDOUT;
```

## Using `SUBSCRIBE` with Python and psycopg2

Setup virtual environment:
```bash
python3 -m venv venv
source venv/bin/activate
pip install psycopg2-binary python-dotenv
```

Run the the `subscribe.py` script:
```bash
python subscribe.py
```

Output:

```py
Waiting for updates...
(Decimal('1729786668000'), False, 1, 90817, 'Alec89', '53188', '41846', 1, None)
(Decimal('1729786670000'), False, 1, 96566, 'Christina_Herzog2', '52301', '71868', 1, None)
(Decimal('1729786672000'), False, 1, 19028, 'Hudson_Heller76', '66648', '24064', 1, None)
(Decimal('1729786674000'), False, 1, 21478, 'Earnest_Ernser', '49801', '48775', 0, None)
(Decimal('1729786676000'), False, 1, 20213, 'Thora_Schamberger', '62960', '67815', 1, None)
```

![simple-example-subscribe](https://github.com/user-attachments/assets/6c92dc54-3cae-4605-ab2a-2885dad0bb86)

## Debugging Materialize

### Check Container Status
```bash
# Check all containers status
docker compose ps

# Check container health
docker compose ps materialized
```

### View Logs
```bash
# View Materialize logs
docker compose logs materialized

# Follow logs in real-time
docker compose logs -f materialized | grep -v 'No such file or directory'

# View last 100 lines
docker compose logs --tail=100 materialized

# Check all services logs
docker compose logs
```

### Resource Management
1. Docker Desktop Resource Limits
   - Open Docker Desktop → Settings → Resources
   - Recommended minimum settings:
     - CPUs: 6
     - Memory: 12GB
     - Swap: 1GB
   - Apply & Restart Docker Desktop

2. Check Resource Usage
```bash
# View container resource usage
docker stats materialized
```

### Check Source Status

If Materialize is not receiving data from the source, you will not see any data when running `SUBSCRIBE` without a snapshot. To check the source status, run the following queries:

```sql
-- Get the source name:
SHOW SOURCES;

-- Check source status:
SELECT * FROM mz_internal.mz_source_statuses
WHERE name = 'SOURCE_NAME';
```

## Cleanup

Stop the services:
```bash
docker compose down -v
```
