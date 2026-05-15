# Just Use Postgres

## Chapter 1: Getting Started / Mock Data

### `generate_series` for Mock Data

```sql
-- Basic series
SELECT generate_series(1, 5);

-- Assign column alias
SELECT generate_series(1, 5) AS id;

-- Random buyer IDs (Postgres 17+)
SELECT
  generate_series(1, 5) AS id,
  random(1, 10) AS buyer_id;

-- Pre-Postgres 17 equivalent
floor(random() * (max_val - min_val + 1)) + min_val

-- Random element from array
SELECT
  generate_series(1, 1000) AS id,
  random(1, 10)             AS buyer_id,
  (ARRAY['AAPL','F','DASH'])[ceil(random() * 3)] AS symbol,
  random(1, 20)             AS order_quantity,
  random(10, 20)            AS bid_price,
  now()                     AS order_time;
```

### Count + GROUP BY pattern

```sql
-- Most traded stocks by volume
SELECT symbol, count(*) AS total_volume
FROM trades
GROUP BY symbol
ORDER BY total_volume DESC;

-- Top 3 buyers by spend
SELECT buyer_id, sum(bid_price * order_quantity) AS total_spend
FROM trades
GROUP BY buyer_id
ORDER BY total_spend DESC
LIMIT 3;
```

---

## Chapter 2: Standard RDBMS Capabilities

### Multi-Tenancy Patterns (3 options)

| Pattern | Description |
|---|---|
| **Table-level** | Shared tables + `tenant_id` column |
| **Schema-level** | Per-tenant schema in one database |
| **Database-level** | Per-tenant database (used in book) |

### Database & Schema Management

```sql
-- Create per-tenant databases
CREATE DATABASE brewery;
CREATE DATABASE coffeechain;

-- List databases
SELECT datname FROM pg_database;

-- Connect (psql meta-command)
\c coffeechain

-- Create schemas
CREATE SCHEMA product;
CREATE SCHEMA customer;
CREATE SCHEMA sales;

-- Check/set current schema
SHOW search_path;
SET search_path TO product;

-- List schemas
\dn
```

### Data Types for IDs

| Type | Description |
|---|---|
| `SERIAL` | 32-bit auto-increment (max ~2.1B) |
| `BIGSERIAL` | 64-bit auto-increment |
| `UUID` | `gen_random_uuid()` → UUID v4 |
| `UUID v7` | Postgres 18+, timestamp-embedded, index-friendly |

### Creating Tables with Constraints

```sql
CREATE TABLE product.catalog (
  id           serial PRIMARY KEY,
  name         text NOT NULL,
  description  text NOT NULL,
  category     text NOT NULL CHECK (category IN ('coffee','mug','t-shirt')),
  price        numeric(10,2) NOT NULL,
  stock_quantity int NOT NULL CHECK (stock_quantity >= 0)
);

CREATE TABLE product.reviews (
  id          bigserial PRIMARY KEY,
  product_id  int,
  customer_id int,
  review      text,
  rank        int
);
```

### Adding Constraints After Creation

```sql
-- Add check constraint
ALTER TABLE product.catalog
  ADD CONSTRAINT catalog_price_check CHECK (price > 0);

-- Add not-null + range constraint on existing table
ALTER TABLE product.reviews
  ADD CONSTRAINT reviews_review_not_null CHECK (review IS NOT NULL),
  ADD CONSTRAINT reviews_rank_range CHECK (rank BETWEEN 1 AND 5);
```

**Warning:** `ALTER TABLE ... ADD CONSTRAINT` fails if existing rows violate the new constraint.

### Foreign Keys

```sql
-- FK on product_id referencing products.catalog(id)
ALTER TABLE product.reviews
  ADD CONSTRAINT fk_product
  FOREIGN KEY (product_id) REFERENCES product.catalog(id);

-- FK on customer_id referencing customers.accounts(id)
ALTER TABLE product.reviews
  ADD CONSTRAINT fk_customer
  FOREIGN KEY (customer_id) REFERENCES customer.accounts(id);

-- Cascade deletes: also deletes child rows
ALTER TABLE product.reviews
  ADD CONSTRAINT fk_customer
  FOREIGN KEY (customer_id) REFERENCES customer.accounts(id)
  ON DELETE CASCADE;
```

**Note:** The referenced column must have a unique constraint or primary key.

### Transactions

```sql
-- Implicit (single statement — always transactional)
UPDATE product.catalog
SET stock_quantity = stock_quantity + 100
WHERE id = 1;

-- Explicit multi-statement
BEGIN;
  INSERT INTO sales.orders (id, customer_id, total_amount)
    VALUES (gen_random_uuid(), 1, 26.53);

  INSERT INTO sales.order_items (order_id, product_id, price)
    VALUES
      ('...uuid...', 1, 16.54),
      ('...uuid...', 4, 9.99);

  UPDATE product.catalog
    SET stock_quantity = stock_quantity - 1
    WHERE id IN (1, 4);
COMMIT;
```

### MVCC and Isolation Levels

- **Default:** `READ COMMITTED` — prevents dirty reads
- **Repeatable Read** — prevents non-repeatable reads
- **Serializable** — prevents phantom reads

Two concurrent `READ COMMITTED` transactions updating the same row will **block** (not overwrite) each other — no dirty writes.

### Exclusion Constraint

```sql
-- Only one pending order per customer at a time
ALTER TABLE sales.orders
  ADD EXCLUDE USING btree (customer_id WITH =)
  WHERE (status = 'pending');
```

### Joins

```sql
-- INNER JOIN: top 3 customers by order count
SELECT c.name, c.id, count(*) AS total_orders
FROM customer.accounts c
JOIN sales.orders s ON c.id = s.customer_id
GROUP BY c.id
ORDER BY total_orders DESC
LIMIT 3;

-- LEFT JOIN: customers with no orders
SELECT c.name
FROM customer.accounts c
LEFT JOIN sales.orders s ON c.id = s.customer_id
WHERE s.id IS NULL;

-- LEFT JOIN: product popularity (nulls last)
SELECT p.name, count(oi.product_id) AS total_sold, p.price
FROM product.catalog p
LEFT JOIN sales.order_items oi ON p.id = oi.product_id
GROUP BY p.id
ORDER BY total_sold DESC NULLS LAST, price DESC;
```

### Functions (PL/pgSQL)

```sql
-- Simple SQL function
CREATE OR REPLACE FUNCTION product.get_product_price(p_id int)
RETURNS numeric AS $$
  SELECT price FROM product.catalog WHERE id = p_id;
$$ LANGUAGE sql;

-- Call it
SELECT product.get_product_price(5);
```

**Key points:**
- Functions execute **atomically and transactionally**
- `RETURNING` clause in DML inside functions lets you capture generated values
- `MERGE` statement for upsert patterns
- `FOR UPDATE SKIP LOCKED` for concurrent queue consumers

### Triggers

```sql
-- Trigger function
CREATE OR REPLACE FUNCTION sales.update_order_total()
RETURNS trigger LANGUAGE plpgsql AS $$
DECLARE
  v_order_id uuid;
  v_total    numeric;
BEGIN
  v_order_id := COALESCE(NEW.order_id, OLD.order_id);
  SELECT COALESCE(sum(quantity * price), 0)
    INTO v_total
    FROM sales.order_items
    WHERE order_id = v_order_id;
  UPDATE sales.orders SET total_amount = v_total WHERE id = v_order_id;
  RETURN NEW;
END;
$$;

-- Attach trigger
CREATE TRIGGER trg_update_order_total
AFTER INSERT OR UPDATE OR DELETE ON sales.order_items
FOR EACH ROW EXECUTE FUNCTION sales.update_order_total();
```

### Listen / Notify

```sql
-- Subscribe
LISTEN queue_new_message;

-- Send notification (with optional payload)
NOTIFY queue_new_message, 'visitor checked in';

-- From PL/pgSQL function
PERFORM pg_notify('queue_new_message', 'visitor checked in');
```

**Limitations:**
- Only subscribers **currently connected** receive notifications (no backlog)
- Not supported on replica nodes — must connect to primary

### Views and Materialized Views

```sql
-- Regular view (executes query every call)
CREATE VIEW sales.product_sales_summary AS
SELECT p.name, sum(oi.price) AS revenue, count(*) AS units_sold
FROM product.catalog p
JOIN sales.order_items oi ON p.id = oi.product_id
GROUP BY p.id;

-- Query the view
SELECT * FROM sales.product_sales_summary
WHERE category = 'coffee';

-- Materialized view (cached result)
CREATE MATERIALIZED VIEW sales.monthly_summary AS
SELECT date_trunc('month', o.created_at) AS month,
       sum(o.total_amount) AS total_sales
FROM sales.orders o
GROUP BY 1;

-- Refresh materialized view
REFRESH MATERIALIZED VIEW sales.monthly_summary;
```

**Tip:** Use `pg_cron` extension or triggers to refresh materialized views automatically.

### Roles and Access Control

```sql
-- Create a role that can log in
CREATE ROLE coffeechain_admin LOGIN;

-- Grant connect to database
GRANT CONNECT ON DATABASE coffeechain TO coffeechain_admin;

-- Revoke public access (PUBLIC = all roles)
REVOKE CONNECT ON DATABASE coffeechain FROM PUBLIC;
REVOKE CONNECT ON DATABASE brewery FROM PUBLIC;

-- Grant schema usage
GRANT USAGE ON SCHEMA public, product, customer, sales
  TO coffeechain_admin;

-- Grant DML on all tables
GRANT SELECT, INSERT, UPDATE, DELETE
  ON ALL TABLES IN SCHEMA product TO coffeechain_admin;
GRANT SELECT, INSERT, UPDATE, DELETE
  ON ALL TABLES IN SCHEMA sales   TO coffeechain_admin;

-- Grant sequence usage (for auto-increment IDs)
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA product
  TO coffeechain_admin;
```

**Key points:**
- `PUBLIC` is a built-in role that includes all users
- Role owners can't drop tables they don't own
- The `postgres` superrole should have limited usage in production

---

## Chapter 3: Modern SQL

### Common Table Expressions (CTEs)

```sql
-- Basic CTE structure
WITH cte_name AS (
  SELECT ...   -- auxiliary statement
)
SELECT ... FROM cte_name;  -- primary statement

-- Multiple CTEs (each can reference previous)
WITH
  plays_cte AS (
    SELECT p.song_id, s.duration, p.play_duration
    FROM streaming.plays p
    JOIN streaming.songs s ON p.song_id = s.id
    WHERE p.play_start_time BETWEEN '2024-09-15' AND '2024-09-16'
  ),
  user_play_counts AS (
    SELECT song_id,
           count(DISTINCT user_id) AS user_count,
           min(play_duration)      AS min_play_duration
    FROM plays_cte
    WHERE play_duration < duration / 2
    GROUP BY song_id
  )
SELECT * FROM user_play_counts WHERE user_count >= 3;
```

**CTE materialization:**
```sql
-- Force materialization (evaluate once, cache result)
WITH cte_name AS MATERIALIZED (SELECT ...)
SELECT ...;

-- Prevent materialization (re-evaluate every reference)
WITH cte_name AS NOT MATERIALIZED (SELECT ...)
SELECT ...;
```

**Rule:** If a CTE is referenced more than once, Postgres materializes it automatically.

### Data-Modifying CTEs

```sql
-- Update inside a CTE, use result in primary query
WITH updated_play AS (
  UPDATE streaming.plays
     SET play_duration = 200
   WHERE id = 30
  RETURNING song_id, play_duration
)
SELECT song_id,
       play_duration = duration AS rank_changed
FROM updated_play
JOIN streaming.songs s ON updated_play.song_id = s.id;
```

**Key behavior:** Multiple data-modifying CTEs in one query execute **concurrently on the same data snapshot** — changes in one CTE are NOT visible to sibling CTEs unless explicitly referenced.

### Recursive Queries

```sql
-- General form
WITH RECURSIVE cte_name (col1, col2, ...) AS (
  -- Non-recursive term (runs once, seeds the working table)
  SELECT ...
  UNION ALL
  -- Recursive term (references cte_name, runs until no rows)
  SELECT ... FROM cte_name WHERE <condition>
)
SELECT * FROM cte_name;

-- Example: walk a song-play sequence chain
WITH RECURSIVE play_sequence (parent_id, song_id, play_start_time) AS (
  -- Non-recursive: start node
  SELECT id, song_id, play_start_time
  FROM streaming.plays
  WHERE id = 5
  UNION ALL
  -- Recursive: follow the chain
  SELECT p.id, p.song_id, p.play_start_time
  FROM streaming.plays p
  JOIN play_sequence ps ON p.played_after = ps.parent_id
)
SELECT * FROM play_sequence ORDER BY play_start_time;

-- With accumulator (level tracking, running total)
WITH RECURSIVE play_sequence (parent_id, sequence, total_duration) AS (
  SELECT id, ARRAY[id], play_duration
  FROM streaming.plays WHERE id = 5
  UNION ALL
  SELECT p.id, ps.sequence || p.id, ps.total_duration + p.play_duration
  FROM streaming.plays p
  JOIN play_sequence ps ON p.played_after = ps.parent_id
)
SELECT * FROM play_sequence;
```

**Tip:** Change `WHERE id = 5` to `WHERE played_after IS NULL` to get all root sequences.

### Window Functions

```sql
-- Basic form: function OVER (PARTITION BY col ORDER BY col)

-- SUM as window function (retains all rows, unlike GROUP BY aggregate)
SELECT user_id, song_id, play_duration,
       sum(play_duration) OVER (PARTITION BY song_id) AS total_duration
FROM streaming.plays;

-- Running total within a window (ORDER BY creates frames)
SELECT user_id, play_duration,
       sum(play_duration) OVER (
         PARTITION BY song_id
         ORDER BY user_id
       ) AS running_total
FROM streaming.plays
WHERE song_id = 2;

-- RANK function
SELECT song_id,
       sum(play_duration) AS total_play_duration,
       rank() OVER (ORDER BY sum(play_duration) DESC) AS song_rank
FROM streaming.plays
GROUP BY song_id
ORDER BY song_rank;
```

**Window-only functions:** `rank()`, `dense_rank()`, `row_number()`, `lag()`, `lead()`, `ntile()`, `first_value()`, `last_value()`

**Key difference from GROUP BY:** Window functions **retain individual rows**; GROUP BY collapses them.

---

## Chapter 4: Indexes

### Index Type Overview

#### By Scope (What to Index)
| Type | Description |
|---|---|
| **Single-column** | One column, B-tree or Hash |
| **Composite** | Multiple columns, order matters |
| **Covering** | Composite + `INCLUDE` for extra columns |
| **Partial** | Subset of rows via `WHERE` clause |
| **Functional/Expression** | Index result of expression/function |

#### By Data Structure (How to Index)
| Type | Use Case |
|---|---|
| **B-tree** | Default; equality + range queries; sortable |
| **Hash** | Equality (`=`, `IN`) only; O(1) lookup |
| **GIN** | Arrays, JSON, full-text search (inverted index) |
| **GiST** | Geometric, full-text, range types |
| **BRIN** | Large tables with correlated physical ordering (e.g. time series) |
| **HNSW** | Vector similarity search (pgVector) |
| **IVF-Flat** | Vector similarity search (pgVector) |
| **Bloom** | Multi-column equality (space-efficient) |
| **RUM** | Like GIN but stores position info for full-text |
| **SPGiST** | Geospatial point data, non-overlapping spaces |

### EXPLAIN / EXPLAIN ANALYZE

```sql
-- Basic plan (no execution)
EXPLAIN SELECT count(*) FROM game.player_stats WHERE win_count > 100;

-- Execute + show actual timing
EXPLAIN ANALYZE SELECT ...;

-- Compact form used throughout book
EXPLAIN (ANALYZE, COSTS OFF) SELECT ...;

-- Full options (for production investigation)
EXPLAIN (ANALYZE, COSTS ON, BUFFERS ON) SELECT ...;
```

**Buffer output meaning:**
- `shared hit = N` → N pages read from shared memory cache
- `read = N` → N pages fetched from disk (slow!)

**Planning time** = time for planner to select execution strategy
**Execution time** = time to actually run the query

**Tip:** Run `ANALYZE` after creating a new index so the planner picks it up.

### B-tree Index

```sql
-- Single column descending (for DESC-ordered leaderboards)
CREATE INDEX idx_score ON game.player_stats (score DESC);

-- Check index definition
SELECT indexdef FROM pg_indexes WHERE indexname = 'idx_score';

-- Access methods the planner may choose:
-- Index Scan       → direct row-by-row via index
-- Index-Only Scan  → all data in index (heap fetches = 0)
-- Bitmap Index Scan + Bitmap Heap Scan → bulk mode for many rows
-- Seq Scan         → full table scan (no usable index)
```

### Hash Index

```sql
-- Hash: O(1), equality only, no range queries
CREATE INDEX idx_champion_title
  ON game.player_stats USING HASH (champion_title);

-- Usable with = and IN operators
SELECT username FROM game.player_stats
WHERE champion_title IN ('Air Force Warlord', 'Naval Warlord');
```

### Composite Index

```sql
-- Column order determines sort order and usability
CREATE INDEX idx_region_score_wincount
  ON game.player_stats (region, score DESC, win_count DESC);

-- Usable for:
--   WHERE region = ?
--   WHERE region = ? AND score > ?
--   WHERE region = ? AND score > ? AND win_count > ?
--   ORDER BY region, score DESC, win_count DESC  (no sort step!)

-- NOT usable (missing leading column):
--   WHERE score > ? AND win_count > ?  (no region)

-- Postgres 18+ supports skip-scan (can skip leading columns in more cases)
```

**Caveat:** Swapping column order in `ORDER BY` prevents index use.

### Covering Index (`INCLUDE`)

```sql
-- Store username in index to enable index-only scans
CREATE INDEX idx_region_covering
  ON game.player_stats (region, score DESC, win_count DESC)
  INCLUDE (username);

-- Result: index-only scan (heap fetches = 0) when querying username
```

**Trade-off:** Included columns must be maintained on every write to that column.

### Partial Index

```sql
-- Index only rows satisfying a condition
CREATE INDEX idx_occasional_players
  ON game.player_stats (playtime)
  WHERE playtime <= INTERVAL '50 hours';

-- Postgres only uses this index when query condition matches the WHERE:
--   playtime <= '50 hours'  → uses index
--   playtime <= '50 hours 1 second' → full scan
```

### Functional / Expression Index

```sql
-- Index the result of an expression
CREATE INDEX idx_performance_margin
  ON game.player_stats ((win_count - loss_count));

-- Query must use the SAME expression to benefit:
SELECT * FROM game.player_stats
WHERE win_count - loss_count BETWEEN 10 AND 50;

-- Expression index on JSON field
CREATE INDEX idx_pizza_type
  ON pizzeria.order_items ((pizza->>'type'));
-- Only usable with double-arrow (->>) not single-arrow (->)
```

### Overindexing Warning

Every index:
- Consumes storage and memory
- Must be maintained on every `INSERT`/`UPDATE`/`DELETE`
- Adds planning overhead

**Strategy:** Use `EXPLAIN` first; aim for minimal indexes that cover multiple query patterns.

---

## Chapter 5: JSON

### JSON vs JSONB

| | `JSON` | `JSONB` |
|---|---|---|
| Storage | Text (original) | Binary (transformed) |
| Write speed | Faster | Slightly slower |
| Read speed | Slower (re-parse) | Faster |
| Key order | Preserved | Not preserved |
| Duplicate keys | Preserved | Removed |
| Operators | Basic | Rich set |
| Indexable (GIN) | No | Yes |

**Recommendation:** Use `JSONB` unless you need to preserve key order.

### JSON Operators

```sql
-- -> returns JSON (keeps quotes for text)
SELECT pizza->'size'  FROM pizzeria.order_items;   -- "large"

-- ->> returns text (strips quotes)
SELECT pizza->>'crust' FROM pizzeria.order_items;  -- thin

-- Chaining for nested fields
SELECT pizza->'toppings'->'veggies' FROM pizzeria.order_items;

-- Array element by index
SELECT pizza->'toppings'->'veggies'->0 FROM pizzeria.order_items;

-- Filtering: arrow returns JSON so compare with JSON string
SELECT * FROM pizzeria.order_items WHERE pizza->'size' = '"small"';
-- Double-arrow returns text so compare without quotes
SELECT * FROM pizzeria.order_items WHERE pizza->>'crust' = 'gluten-free';

-- Key existence (JSONB only)
SELECT * FROM pizzeria.order_items WHERE pizza ? 'special_instructions';

-- Containment: left side contains right side (JSONB only)
SELECT count(*) FROM pizzeria.order_items
WHERE pizza @> '{"crust": "gluten-free"}';

-- Combined containment
SELECT count(*) FROM pizzeria.order_items
WHERE pizza @> '{"crust":"gluten-free","type":"custom"}';
```

### JSON Path Expressions

```sql
-- jsonb_path_query: extract values matching a path
SELECT jsonb_path_query(pizza, '$.type') FROM pizzeria.order_items;

-- Array elements: [*] operator
SELECT jsonb_path_query(pizza, '$.toppings.cheese[*]')
FROM pizzeria.order_items;

-- Filter within path: ? operator
-- @ = current element in iteration
SELECT count(*) FROM pizzeria.order_items
WHERE jsonb_path_exists(pizza, '$.toppings.cheese[*] ? (@.parmesan != null)');

-- Chained filters
SELECT count(*) FROM pizzeria.order_items
WHERE jsonb_path_exists(pizza,
  '$ ? (@.type == "custom").toppings.cheese[*].parmesan ? (@ == "extra")');
```

### Modifying JSON

```sql
-- jsonb_set(original, path_array, new_value, create_if_missing)
UPDATE pizzeria.order_items
SET pizza = jsonb_set(pizza, '{crust}', '"regular"', false)
WHERE order_id = 1 AND order_item_id = 1;

-- Update nested array
UPDATE pizzeria.order_items
SET pizza = jsonb_set(pizza, '{toppings,veggies}',
  '[{"tomato":"extra"},{"spinach":"regular"}]', false)
WHERE order_id = 1;

-- Update specific array element by index
UPDATE pizzeria.order_items
SET pizza = jsonb_set(pizza, '{toppings,veggies,0,tomato}', '"extra"', false)
WHERE order_id = 1;

-- Delete a key (# - operator)
UPDATE pizzeria.order_items
SET pizza = pizza #- '{toppings,meats}'
WHERE order_id = 1;
```

### JSON Indexes

```sql
-- Expression index on a specific JSON field (B-tree)
CREATE INDEX idx_pizza_type ON pizzeria.order_items ((pizza->>'type'));
-- Only works for the exact expression used in queries

-- Default GIN index (indexes all keys, values, array elements)
CREATE INDEX idx_pizza_orders_gin
  ON pizzeria.order_items USING GIN (pizza);
-- Supports: ?, @>, @?, @@  operators

-- Non-default GIN with jsonb_path_ops (indexes paths+values, uses hashing)
CREATE INDEX idx_pizza_orders_pathops_gin
  ON pizzeria.order_items USING GIN (pizza jsonb_path_ops);
-- Supports: @>, @?, @@  (NOT ?)
-- Smaller, faster for containment; doesn't support key-existence checks
```

**Comparison:**

| | Default GIN (`jsonb_ops`) | Path GIN (`jsonb_path_ops`) |
|---|---|---|
| Size | Larger | Smaller (~50%) |
| Lookup speed | Good | Better for `@>` |
| Supports `?` | Yes | No |
| Supports `@>` | Yes | Yes |

---

## Chapter 6: Full-Text Search

### How It Works (Overview)

1. **Tokenize** — split text into tokens
2. **Normalize** — stem tokens, remove stop words → lexemes stored as `tsvector`
3. **Store/Index** — persist lexemes in a `tsvector` column; index with GIN or GiST
4. **Search** — convert user query to `tsquery`, use `@@` match operator

### Tokenization and Normalization

```sql
-- Debug tokenization/normalization
SELECT * FROM ts_debug('english', 'Five explorers are traveling to a distant galaxy');

-- Simple conversion to tsvector
SELECT to_tsvector('english', 'Five explorers are traveling to a distant galaxy');
-- Returns: '5':1 'explor':2 'galaxi':8 'travel':4 'distant':7
-- Note: 'are','to','a' removed as stop words; stems applied

-- Using regconfig explicitly (required for generated columns)
SELECT to_tsvector('english', 'Space Odyssey ' || description)
FROM movies;
```

### Full-Text Search Configurations

```sql
-- Check default config
SHOW default_text_search_config;  -- usually 'english'

-- List available configs
\dF

-- Use a specific language
SELECT to_tsvector('russian', 'Пять исследователей путешествуют к далёкой галактике');
```

### Storing Lexemes as Generated Column

```sql
-- Add a stored generated column (auto-updates when source columns change)
ALTER TABLE omdb.movies
  ADD COLUMN lexemes tsvector
    GENERATED ALWAYS AS (
      setweight(to_tsvector('english', coalesce(name, '')), 'A') ||
      setweight(to_tsvector('english', coalesce(description, '')), 'B')
    ) STORED;
-- Note: Must use explicit config string in generated columns (immutable requirement)
```

### Querying with tsquery

```sql
-- plainto_tsquery: all words must be present (AND operator)
SELECT plainto_tsquery('english', 'a computer animated film');
-- Returns: 'comput' & 'anim' & 'film'

-- Full-text search with @@ operator
SELECT id, name FROM omdb.movies
WHERE lexemes @@ plainto_tsquery('english', 'a computer animated film');

-- to_tsquery: explicit operators (&, |, !, <->)
SELECT to_tsquery('english', 'computer & animated & (lion | clownfish | donkey)');

-- Phrase search with <->  (followed-by operator)
SELECT to_tsquery('english', 'the <-> lion <-> king');

-- Distance operator: <N>  (N tokens apart)
SELECT to_tsquery('english', 'return <3> king');

-- phrase_to_tsquery: like plainto but inserts <-> between words
SELECT phrase_to_tsquery('english', 'star wars');

-- websearch_to_tsquery: handles raw user input safely
SELECT websearch_to_tsquery('english', 'star wars -"the force"');
```

### Ranking Results

```sql
-- ts_rank: frequency-based ranking
SELECT id, name,
       ts_rank(lexemes, to_tsquery('english', 'ghosts')) AS rank
FROM omdb.movies
WHERE lexemes @@ to_tsquery('english', 'ghosts')
ORDER BY rank DESC, vote_average DESC;

-- ts_rank with normalization (1 = divide by document length)
SELECT ts_rank(lexemes, query, 1) AS rank ...

-- setweight: assign label A (high) or B (low) to lexeme parts
SELECT setweight(to_tsvector('english', name), 'A') ||
       setweight(to_tsvector('english', description), 'B')
FROM omdb.movies WHERE id = 251;
-- Lexemes from name get 'A' label; description gets 'B'
-- Default weights: D=0.1, C=0.2, B=0.4, A=1.0

-- Filter by weight in tsquery
SELECT * FROM omdb.movies
WHERE lexemes @@ to_tsquery('english', 'ghosts:A');
-- Only matches if 'ghost' lexeme has 'A' label (i.e., in title)
```

### Highlighting Results

```sql
-- ts_headline: returns fragments with search terms highlighted
SELECT ts_headline(
  'english',
  description,
  to_tsquery('english', 'pirates'),
  'MaxFragments=3, MinWords=5, MaxWords=10, FragmentDelimiter=» «'
)
FROM omdb.movies
WHERE lexemes @@ to_tsquery('english', 'pirates:B')
ORDER BY ts_rank(lexemes, to_tsquery('english', 'pirates')) DESC
LIMIT 1;
-- Default highlight markers: <b>word</b>
-- WARNING: output is not XSS-safe for HTML-containing documents
```

### Indexing Lexemes

```sql
-- GIN index on tsvector column (preferred — faster lookups)
CREATE INDEX idx_movie_lexemes_gin ON omdb.movies USING GIN (lexemes);

-- GiST index (smaller, faster to build/update, slower lookups)
CREATE INDEX idx_movie_lexemes_gist ON omdb.movies USING GIST (lexemes);
```

**GIN vs GiST for Full-Text:**

| | GIN | GiST |
|---|---|---|
| Lookup speed | Faster | Slower (signature collisions → recheck) |
| Index size | Larger | Smaller |
| Build/update time | Slower | Faster |

**Default signature length for GiST:** 124 bytes; max 2024 bytes (longer = fewer false positives but larger index).

**Note:** GIN doesn't store lexeme positions — use RUM extension if positional queries are frequent.

---

## Chapter 7: Extensions

### Extension Management

```sql
-- List all available extensions on server
SELECT * FROM pg_available_extensions;

-- Check specific extension
SELECT * FROM pg_available_extensions WHERE name = 'pgcrypto';

-- Install/enable for current database
CREATE EXTENSION pgcrypto;

-- Verify installed extensions
SELECT * FROM pg_extension;
-- or
\dx

-- Disable extension
DROP EXTENSION pgcrypto;
```

### pgCrypto — Password Hashing

```sql
-- Hash a password when creating account
INSERT INTO accounts (username, password_hash)
VALUES ('a_hamilton', crypt('SuperSecret123', gen_salt('bf')));

-- Verify password on login
SELECT username FROM accounts
WHERE username = 'a_hamilton'
  AND password_hash = crypt('SuperSecret123', password_hash);
-- Returns row if match, empty if wrong password
```

**Key points:**
- `gen_salt('bf')` generates a random Blowfish salt per password
- `crypt()` stores the salt inside the hash value — used for both hashing and verification
- If bad actors get the DB, they can't see plain-text passwords

### Extension Categories

| Category | Examples |
|---|---|
| Beyond Relational | pgVector, pgai, TimescaleDB, PostGIS, pgMQ, pgDuckDB |
| Procedural Languages | PLV8 (JavaScript), PLJava, PLPython, PLRust |
| Foreign Data Wrappers | file_fdw, postgres_fdw, mysql_fdw, redis_fdw, parquet_s3_fdw, kafka_fdw |
| Query/Performance | pg_stat_statements, auto_explain, HypoPG |
| Tools & Utilities | pg_cron, postgresql_anonymizer, pgaudit, pgpartman |

### Postgres-Compatible Solutions

- **Built from scratch** (support Postgres wire protocol): Google Spanner, CockroachDB
- **Built on Postgres source**: Neon (serverless), YugabyteDB (distributed), Citus Data (sharding)

---

## Chapter 8: Generative AI / pgVector

### Setup

```sql
-- Enable pgVector extension
CREATE EXTENSION vector;

-- Define a vector column (1024-dim example)
ALTER TABLE omdb.movies ADD COLUMN movie_embedding vector(1024);
```

### Distance Operators

| Operator | Distance Type | Notes |
|---|---|---|
| `<=>` | Cosine distance | Common for text embeddings, especially L2-normalized |
| `<->` | Euclidean (L2) distance | |
| `<#>` | Negative inner product | |

### Vector Similarity Search

```sql
-- Find 3 most similar movies to an embedding
SELECT name, description
FROM omdb.movies
ORDER BY movie_embedding <=> (
  SELECT phrase_embedding FROM omdb.phrases WHERE phrase = 'May the Force Be With You'
)
LIMIT 3;

-- With cosine distance in output
WITH phrase AS (
  SELECT phrase_embedding AS emb FROM omdb.phrases WHERE phrase = 'A Movie About a Jedi'
)
SELECT m.name,
       m.movie_embedding <=> p.emb AS cosine_distance
FROM omdb.movies m, phrase p
ORDER BY cosine_distance
LIMIT 3;
```

**Note:** Distance ranges from 0 (identical) to 1 (completely different) for cosine distance with normalized vectors.

### IVF-Flat Index

```sql
-- Create IVF-Flat index
CREATE INDEX idx_movie_embeddings_ivfflat
  ON omdb.movies USING ivfflat (movie_embedding vector_cosine_ops)
  WITH (lists = 5);
-- lists rule: rows/1000 (for ≤1M rows), sqrt(rows) (for >1M rows)

-- Distance operator in queries MUST match index definition (vector_cosine_ops)

-- Increase probes for better recall (within transaction)
BEGIN;
  SET LOCAL ivfflat.probes = 2;
  SELECT name FROM omdb.movies ORDER BY movie_embedding <=> $1 LIMIT 3;
COMMIT;
```

**IVF-Flat characteristics:**
- Divides embeddings into `lists` clusters by k-means
- At query time, searches the `probes` nearest cluster centroids
- Centroids are fixed at build time — needs rebuild if data changes significantly
- Better for static datasets; smaller index size vs HNSW

### HNSW Index

```sql
-- Create HNSW index
CREATE INDEX idx_movie_embeddings_hnsw
  ON omdb.movies USING hnsw (movie_embedding vector_cosine_ops)
  WITH (m = 8, ef_construction = 16);
-- m: max connections per node (default 16); higher = denser graph, better recall, slower build
-- ef_construction: candidate list size during build (default 64); higher = better index quality

-- Tune search accuracy at query time
SET hnsw.ef_search = 50;  -- session level (default 40)
-- or
BEGIN;
  SET LOCAL hnsw.ef_search = 50;  -- single query
  SELECT name FROM omdb.movies ORDER BY movie_embedding <=> $1 LIMIT 3;
COMMIT;
```

**HNSW characteristics:**
- Multi-layer graph: sparse upper layers, dense lower layers
- Stable recall even as data grows (nodes added to existing graph)
- Larger index, longer build time vs IVF-Flat
- **Preferred for growing datasets or when recall consistency matters**

### IVF-Flat vs HNSW

| | IVF-Flat | HNSW |
|---|---|---|
| Index size | Smaller | Larger |
| Build time | Faster | Slower |
| Recall stability with updates | Lower | Higher |
| Best for | Static, snapshot datasets | Growing datasets |

### Approximate vs Exact Nearest Neighbor

- **ENN (exact):** full table scan, recall = 1.0
- **ANN (approximate):** index scan, recall < 1.0 (tunable)
- Recall measured 0–1; 1 = perfect (all true nearest neighbors returned)

### RAG Architecture (conceptual)

```
User prompt → Embedding model → vector
  → Postgres similarity search → top-K context documents
  → LLM with context injected → final response
```

**Key extensions:**
- `pgVector` — vector storage + search
- `pgVectorScale` — adds Streaming DiskANN index
- `pgai` — generate embeddings in SQL, full RAG workflow in database

---

## Chapter 9: Time Series (TimescaleDB)

### Time Series Best Practices

- Every measurement must include a timestamp
- Data is **append-only** (never update historical records)
- Partition by time to keep recent queries fast
- Separate current vs historical data

### Table Partitioning (Core Postgres)

```sql
-- Range partition parent table
CREATE TABLE watch.heart_rate_measurements (
  watch_id    bigint    NOT NULL,
  recorded_at timestamptz NOT NULL,
  heart_rate  int       NOT NULL,
  activity    text
) PARTITION BY RANGE (recorded_at);

-- Create monthly partitions
CREATE TABLE measurements_jan_2025
  PARTITION OF watch.heart_rate_measurements
  FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE measurements_feb_2025
  PARTITION OF watch.heart_rate_measurements
  FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- Postgres transparently routes queries to correct partitions
EXPLAIN SELECT * FROM watch.heart_rate_measurements
WHERE recorded_at BETWEEN '2025-01-01' AND '2025-01-03';
-- Only scans measurements_jan_2025
```

**Note:** Bounds are inclusive on lower end, exclusive on upper.

### TimescaleDB Hypertable

```sql
-- Convert regular table to hypertable (auto-manages partitions)
SELECT create_hypertable(
  'watch.heart_rate_measurements',
  by_range('recorded_at', INTERVAL '1 month'),
  create_default_indexes => false
);

-- View chunks (partitions)
SELECT show_chunks('watch.heart_rate_measurements');

-- Inspect a chunk
\d _timescaledb_internal._hyper_1_1_chunk

-- Auto-creates new chunk when data outside current range is inserted
```

### Data Retention Policy

```sql
-- Drop data older than 30 days automatically
SELECT add_retention_policy(
  'watch.heart_rate_measurements',
  INTERVAL '30 days'
);

-- Custom schedule (every 12 hours)
SELECT add_retention_policy(
  'watch.heart_rate_measurements',
  INTERVAL '30 days',
  schedule_interval => INTERVAL '12 hours'
);
```

### time_bucket Function

```sql
-- Average and max heart rate in 10-minute intervals during workout
SELECT
  time_bucket('10 minutes', recorded_at) AS period,
  avg(heart_rate) AS avg_rate,
  max(heart_rate) AS max_rate
FROM watch.heart_rate_measurements
WHERE watch_id = 1
  AND activity = 'workout'
  AND recorded_at >= '2025-04-23'
  AND recorded_at < '2025-04-24'
GROUP BY period
ORDER BY period;

-- Weekly summary with explicit origin (align to start of month)
SELECT
  time_bucket('1 week', recorded_at,
    origin => '2025-04-01'::timestamptz) AS period,
  activity,
  avg(heart_rate)
FROM watch.heart_rate_measurements
WHERE watch_id = 1
  AND recorded_at BETWEEN '2025-04-01' AND '2025-04-15'
GROUP BY period, activity
ORDER BY period;

-- User-specific timezone
BEGIN;
  SET LOCAL TIME ZONE 'Asia/Tokyo';
  SELECT time_bucket('1 week', recorded_at, timezone => 'Asia/Tokyo') AS period,
         avg(heart_rate)
  FROM watch.heart_rate_measurements
  WHERE watch_id = 2
    AND recorded_at BETWEEN '2025-04-01' AND '2025-04-15'
  GROUP BY period;
COMMIT;
```

### time_bucket_gapfill

```sql
-- Fill in missing time buckets
SELECT
  time_bucket_gapfill('1 minute', recorded_at) AS period,
  avg(heart_rate) AS avg_rate
FROM watch.heart_rate_measurements
WHERE watch_id = 1
  AND recorded_at BETWEEN '2025-01-01 07:25' AND '2025-01-01 07:40'
GROUP BY period;

-- Fill gaps with LOCF (Last Observation Carried Forward)
SELECT
  time_bucket_gapfill('1 minute', recorded_at) AS period,
  locf(avg(heart_rate)) AS avg_rate
FROM watch.heart_rate_measurements ...
GROUP BY period;

-- Fill gaps with linear interpolation
SELECT
  time_bucket_gapfill('1 minute', recorded_at) AS period,
  interpolate(avg(heart_rate)) AS avg_rate
FROM watch.heart_rate_measurements ...
GROUP BY period;
```

### Continuous Aggregates

```sql
-- Create continuous aggregate (special materialized view stored in hypertable)
CREATE MATERIALIZED VIEW watch.low_heart_rate_per_5min
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('5 minutes', recorded_at) AS bucket,
  watch_id,
  min(heart_rate)                       AS min_rate,
  count(*) FILTER (WHERE heart_rate < 50) AS low_rate_count,
  count(*)                              AS total_measurements
FROM watch.heart_rate_measurements
GROUP BY bucket, watch_id;

-- Query like a regular table
SELECT bucket, low_rate_count,
       low_rate_count >= 5 AS bradycardia_detected
FROM watch.low_heart_rate_per_5min
WHERE watch_id = 1
  AND bucket = '2025-11-30 02:35';

-- Define automatic refresh policy
SELECT add_continuous_aggregate_policy(
  'watch.low_heart_rate_per_5min',
  start_offset => INTERVAL '15 minutes',
  end_offset   => INTERVAL '1 minute',
  schedule_interval => INTERVAL '1 minute'
);

-- Manual refresh
CALL refresh_continuous_aggregate(
  'watch.low_heart_rate_per_5min',
  '2025-12-01 00:45', '2025-12-01 00:50'
);

-- Retention policy on aggregate
SELECT add_retention_policy('watch.low_heart_rate_per_5min', INTERVAL '7 days');
```

### Time Series Indexes

```sql
-- B-tree composite index (leading column = timestamp)
CREATE INDEX idx_heart_rate_btree
  ON watch.heart_rate_measurements (recorded_at, watch_id);
-- Usable for: WHERE recorded_at = ?
--             WHERE recorded_at = ? AND watch_id = ?
-- NOT usable (Postgres ≤17): WHERE watch_id = ?  (skips leading column)
-- Postgres 18+: skip-scan allows skipping leading columns

-- BRIN index (much smaller; works because timestamps are physically ordered)
CREATE INDEX idx_heart_rate_brin
  ON watch.heart_rate_measurements USING BRIN (recorded_at);

-- Compare sizes
SELECT indexname,
       pg_size_pretty(pg_relation_size(indexname::regclass)) AS size
FROM pg_indexes
WHERE tablename LIKE '%heart_rate%';
```

**B-tree vs BRIN for time series:**

| | B-tree | BRIN |
|---|---|---|
| Size | Large | Tiny (~100x smaller) |
| Lookup accuracy | Exact row pointers | Page ranges only (may recheck) |
| Best for | Small to medium data, exact range queries | Very large tables with correlated ordering |

---

## Chapter 10: Geospatial (PostGIS)

### PostGIS Data Types

| Type | Description |
|---|---|
| `geometry` | Euclidean plane; Cartesian math; fast but less accurate for large distances |
| `geography` | Spherical model (Earth's curvature); accurate; default SRID 4326 (WGS84/GPS) |

```sql
-- geometry point in Web Mercator (SRID 3857, meters)
location geometry(Point, 3857)

-- geography point in WGS84 (SRID 4326, lat/lon)
location geography(Point, 4326)

-- Insert geometry point (Web Mercator)
ST_SetSRID(ST_MakePoint(-8381068.48, 2970863.11), 3857)

-- Insert geography point (WGS84: longitude, latitude)
ST_GeographyFromText('POINT(-80.1918 25.7742)')
```

### Common PostGIS Functions

```sql
-- Transform coordinates between SRIDs
ST_Transform(geom, 4326)         -- convert to WGS84 lat/lon

-- Extract X (longitude) and Y (latitude)
ST_X(geom), ST_Y(geom)

-- Distance between two geometries (in SRID units)
ST_Distance(g1, g2)

-- Check if within distance (INDEX-AWARE → use this not ST_Distance in WHERE)
ST_DWithin(hotel.way, place.way, 500)

-- Create a point from coordinates
ST_SetSRID(ST_MakePoint(x, y), 3857)

-- Number of points in a geometry
ST_NPoints(geom)

-- Area of polygon (in SRID units — meters² for SRID 3857)
ST_Area(geom)

-- Length of line string
ST_Length(geom)

-- Check if A is completely within B
ST_Within(geomA, geomB)

-- Check if geometries share at least one point
ST_Intersects(geomA, geomB)

-- Check if geometries are topologically equal
ST_Equals(geomA, geomB)

-- Expand geometry by distance (creates bounding box)
ST_Expand(geom, distance_meters)

-- Convert binary WKB to human-readable WKT
ST_AsText(geom)

-- Text representation WKT with SRID
ST_AsEWKT(geom)
```

### Finding Nearby Points

```sql
-- Find restaurants within 500m of a hotel
WITH hotel AS (
  SELECT way FROM florida.planet_osm_point
  WHERE name = 'Roost Apartment Hotel'
)
SELECT p.name,
       round(ST_Distance(h.way, p.way))    AS dist_m,
       round(ST_Distance(h.way, p.way) * 3.28084) AS dist_ft
FROM florida.planet_osm_point p, hotel h
WHERE ST_DWithin(h.way, p.way, 500)
  AND p.amenity = 'restaurant'
ORDER BY dist_m
LIMIT 5;
```

### Polygon Containment

```sql
-- Find parks inside Walt Disney World
WITH disney AS (
  SELECT way AS boundaries
  FROM florida.planet_osm_polygon
  WHERE name = 'Walt Disney World'
)
SELECT p.name
FROM florida.planet_osm_polygon p, disney d
WHERE ST_Within(p.way, d.boundaries)
  AND p.tourism = 'theme_park'
  AND p.name IS NOT NULL
  AND NOT ST_Equals(p.way, d.boundaries);  -- exclude Walt Disney World itself
```

### Spatial Indexes for PostGIS

```sql
-- GiST index (default, general purpose)
CREATE INDEX idx_planet_osm_point_way
  ON florida.planet_osm_point USING GIST (way)
  WITH (fillfactor = 100);

-- SPGiST (better for uniformly distributed point data, no overlaps)
-- BRIN (very large datasets with spatial correlation)
```

**Index-aware functions** (can use GiST): `ST_DWithin`, `ST_Contains`, `ST_Intersects`, `ST_Overlaps`, `ST_Within`

**NOT index-aware**: `ST_Distance` (always computes exact distance — use `ST_DWithin` in `WHERE`, then `ST_Distance` for display)

---

## Chapter 11: Message Queue

### When to Use Postgres as a Queue

1. You need **atomic** enqueue + business operation (same transaction)
2. Your message **volume fits** within Postgres capacity
3. You **already use Postgres** and want to avoid adding another system

### Custom Queue Implementation

```sql
-- Schema + type + table
CREATE SCHEMA mq;
CREATE TYPE mq.message_status AS ENUM ('new', 'processing', 'completed');

CREATE TABLE mq.q (
  id         bigserial    PRIMARY KEY,
  message    json         NOT NULL,
  created_at timestamptz  NOT NULL DEFAULT now(),
  status     mq.message_status NOT NULL DEFAULT 'new'
);

-- enqueue function
CREATE OR REPLACE FUNCTION mq.enqueue(new_message json)
RETURNS void LANGUAGE plpgsql AS $$
BEGIN
  INSERT INTO mq.q (message) VALUES (new_message);
END;
$$;

-- dequeue function (FOR UPDATE SKIP LOCKED for concurrency)
CREATE OR REPLACE FUNCTION mq.dequeue(messages_cnt int)
RETURNS TABLE (id bigint, message json, created_at timestamptz)
LANGUAGE plpgsql AS $$
BEGIN
  RETURN QUERY
  WITH new_messages AS (
    SELECT q.id FROM mq.q
    WHERE status = 'new'
    ORDER BY created_at ASC
    LIMIT messages_cnt
    FOR UPDATE SKIP LOCKED
  )
  UPDATE mq.q
  SET status = 'processing'
  WHERE mq.q.id IN (SELECT nm.id FROM new_messages)
  RETURNING mq.q.id, mq.q.message, date_trunc('second', mq.q.created_at);
END;
$$;

-- markCompleted function
CREATE OR REPLACE FUNCTION mq.mark_completed(
  message_ids bigint[],
  to_delete   boolean DEFAULT false
)
RETURNS void LANGUAGE plpgsql AS $$
BEGIN
  IF to_delete THEN
    DELETE FROM mq.q WHERE id = ANY(message_ids);
  ELSE
    UPDATE mq.q SET status = 'completed' WHERE id = ANY(message_ids);
  END IF;
END;
$$;
```

### `FOR UPDATE SKIP LOCKED`

```sql
-- Multiple consumers can run concurrently without blocking each other
-- Each consumer gets a different set of rows
SELECT id FROM mq.q
WHERE status = 'new'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

### Queue Indexes for High Throughput

```sql
-- Single column on created_at (fast FIFO retrieval)
CREATE INDEX idx_mq_created_at ON mq.q (created_at);

-- Partial index on only 'new' messages (compact + efficient)
CREATE INDEX idx_mq_new_messages
  ON mq.q (created_at, status)
  WHERE status = 'new';
```

### Queue Partitioning Strategy

```sql
-- Partition parent table by creation time
CREATE TABLE mq.q (
  id         bigserial   NOT NULL,
  message    json        NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  status     text        NOT NULL DEFAULT 'new'
) PARTITION BY RANGE (created_at);

-- Create daily partitions
CREATE TABLE mq.q_2025_06_22
  PARTITION OF mq.q
  FOR VALUES FROM ('2025-06-22') TO ('2025-06-23');

CREATE TABLE mq.q_default PARTITION OF mq.q DEFAULT;
```

### DEAD TUPLES / VACUUM Note

- `UPDATE` creates a dead tuple (old row version) + new row
- `DELETE` also leaves dead tuples
- Postgres `autovacuum` eventually removes them
- For high-throughput queues: tune `autovacuum` aggressively or use partitioning to `DROP` old partitions outright

### PGMQ Extension

```sql
-- Enable
CREATE EXTENSION pgmq;

-- Create a named queue
SELECT pgmq.create('visitors_queue');

-- Enqueue
SELECT pgmq.send('visitors_queue', '{"name": "Emily Carter", "service": "car_registration"}');

-- Pop (retrieve + delete immediately)
SELECT * FROM pgmq.pop('visitors_queue');

-- Read with visibility timeout (120 seconds)
SELECT * FROM pgmq.read(
  queue_name => 'visitors_queue',
  vt         => 120,
  qty        => 1
);
-- Message is invisible to other consumers for 120s
-- If not archived/deleted within timeout, becomes visible again

-- Archive (move to archive table)
SELECT pgmq.archive('visitors_queue', msg_id);

-- Query archive
SELECT * FROM pgmq.a_visitors_queue;
```

**PGMQ key feature:** Visibility timeout (`vt`) supports automatic failover — if consumer crashes, message reappears after timeout.

---

## Appendix A: Top 5 Optimization Tips

### 1. Master EXPLAIN

```sql
-- Full diagnostic (for investigating slow queries)
EXPLAIN (ANALYZE, COSTS ON, BUFFERS ON) SELECT ...;

-- Postgres 18+: buffers included by default
-- Postgres 17 and earlier: must add BUFFERS ON explicitly

-- Key metrics:
-- Buffers shared hit = N   → served from RAM (fast)
-- Buffers read = N         → fetched from disk (slow — consider more RAM)
-- Planning time            → time spent choosing execution plan
-- Execution time           → actual query run time
-- Width = 0 in COUNT(*)    → no column data fetched (optimization active)
```

### 2. Index Strategy: Explain → Underindexing → Overindexing

- **Always check EXPLAIN before adding an index**
- Underindexing: full table scan on large table → create appropriate index
- Overindexing: too many indexes → write amplification, slower planning
- One well-designed GIN/composite index can serve many query patterns

### 3. Connection Pooling

- Postgres spawns a new OS process per connection (expensive at scale)
- Use **client-side poolers**: built into language drivers/frameworks
- Use **server-side poolers**: PgBouncer, pgPool, Odyssey
- Managed cloud services typically include a pooler
- Controls max concurrent queries, prevents CPU/memory exhaustion

### 4. Select Only What You Need

```sql
-- BAD (fetches all columns, wastes network + memory)
SELECT * FROM product.catalog WHERE category = 'coffee';

-- GOOD (fetch only needed columns)
SELECT id, name, price FROM product.catalog WHERE category = 'coffee';

-- EXCEPTION: COUNT(*) is optimized — does NOT fetch column data
SELECT count(*) FROM product.catalog;
-- vs
SELECT count(id) FROM product.catalog;  -- fetches id column (4 bytes per row)
```

### 5. Use Postgres Computational Capabilities

- `ORDER BY` in SQL > sorting in application code
- `GROUP BY` + aggregates > looping in application code
- Window functions > self-joins or multiple round trips
- PL/pgSQL functions for complex, data-intensive business logic that would require multiple round trips
- CTEs for breaking complex queries into readable steps

---

## Key Postgres-Specific Behaviors & Non-Obvious Notes

### MVCC (Multi-Version Concurrency Control)

- Rolled-back changes remain physically in the table until vacuumed
- `READ COMMITTED` (default): no dirty reads, concurrent reads don't block writers
- Updating a row creates a new version + dead tuple (old version)
- `autovacuum` reclaims dead tuples

### Timestamps

- `timestamptz` stores as UTC microseconds since 2000-01-01
- Displayed in session/database time zone on read
- Use `SHOW TIME ZONE` to check current setting

### UUID Generation

```sql
-- UUID v4 (random)
SELECT gen_random_uuid();

-- UUID v7 (Postgres 18+, timestamp-embedded, index-friendly)
SELECT uuidv7();
```

### Sequences vs UUIDs

- Sequences require lock contention under heavy concurrency
- UUIDs (`gen_random_uuid()`) are lock-free for high-concurrency inserts
- UUID v7 is index-friendlier than UUID v4 because temporally close values are stored near each other

### `DISTINCT ON`

```sql
-- Return first row per unique value (ordered by specified sort)
SELECT DISTINCT ON (region) region, username, score
FROM game.player_stats
ORDER BY region, score DESC, win_count DESC;
```

### `RETURNING` Clause

```sql
-- Capture auto-generated values after insert
INSERT INTO sales.orders DEFAULT VALUES RETURNING id;

-- Use in CTE
WITH new_order AS (
  INSERT INTO sales.orders (customer_id) VALUES (1) RETURNING id
)
INSERT INTO sales.order_items (order_id, product_id)
SELECT new_order.id, 5 FROM new_order;
```

### `MERGE` Statement (Postgres 15+)

```sql
-- Upsert: insert or update based on match
MERGE INTO sales.order_items AS target
USING (VALUES (order_id_val, product_id_val, qty_val)) AS source(order_id, product_id, qty)
ON target.order_id = source.order_id AND target.product_id = source.product_id
WHEN MATCHED THEN
  UPDATE SET quantity = target.quantity + source.qty
WHEN NOT MATCHED THEN
  INSERT (order_id, product_id, quantity) VALUES (source.order_id, source.product_id, source.qty);
```

### Partition Key in Composite Index

- In Postgres ≤17: leading column must appear in query for composite index to be used
- In Postgres 18+: **skip-scan** allows skipping leading columns

### `timestamptz` Arithmetic

```sql
SELECT now() - INTERVAL '30 minutes';
SELECT date_trunc('month', now());
SELECT extract(epoch from now());
```

### `COALESCE`

```sql
-- Return first non-null value
SELECT COALESCE(null, null, 'fallback');  -- 'fallback'
SELECT COALESCE(sum(quantity * price), 0) FROM order_items WHERE order_id = ...;
```

### `pg_notify` vs `NOTIFY`

```sql
-- In plain SQL
NOTIFY channel_name, 'payload';

-- In PL/pgSQL functions (preferred)
PERFORM pg_notify('channel_name', 'payload');
```

### Table Partitioning Notes

- Each partition is a real Postgres table (can be inspected with `\d`)
- Query the parent table; Postgres routes to correct partition automatically (**partition pruning**)
- Indexes created on parent are automatically created on each child partition
- `DROP TABLE partition_name` is instant (vs `DELETE` which leaves dead tuples)

---

## Extensions Quick Reference

| Extension | Purpose | Chapter |
|---|---|---|
| `pgcrypto` | Cryptographic functions (password hashing, encryption) | 7 |
| `pgvector` | Vector embeddings + HNSW/IVF-Flat indexes | 8 |
| `timescaledb` | Time series: hypertables, continuous aggregates, gap-fill | 9 |
| `postgis` | Geospatial: geometry/geography types, spatial functions | 10 |
| `pgmq` | Message queue with visibility timeouts, API parity with AWS SQS | 11 |
| `pgcron` | Cron-based job scheduler inside Postgres | 7, 9, 11 |
| `pgpartman` | Automated partition creation and maintenance | 9, 11 |
| `pgtrgm` | Trigram matching for fuzzy/spell-check search | 6 |
| `pg_stat_statements` | Track SQL execution statistics | 7 |
| `auto_explain` | Auto-log plans for slow queries | 7 |
| `hypopg` | Hypothetical indexes (test without building) | 7 |
| `pgaudit` | Detailed audit logging | 7 |
| `rum` | Like GIN but stores lexeme positions (faster phrase queries) | 6 |
| `pgduckdb` | Embedded DuckDB columnar engine for analytics | 7 |
| `pgai` | Generate embeddings and run RAG workflows in SQL | 8 |
| `plv8` | JavaScript functions in Postgres (V8 engine) | 7 |
| `pgrouting` | Route finding, driving distances | 10 |
