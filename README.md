# What's streaming on Twitch?

<p align="center">
<img width="600" alt="demo_overview" src="https://user-images.githubusercontent.com/23521087/130613715-5cd0aa0e-a2cc-4bc5-aa42-8309a77a8895.png">
</p>

## Docker

#### Getting the setup up and running

```bash

# Start the setup
docker-compose build

docker-compose up -d

# Is everything really up and running?
docker-compose ps
```

## Kafka

Check that the `twitch-streams` topic has been created:

```bash
docker-compose exec kafka kafka-topics --list --bootstrap-server kafka:9092
```

And that there's data landing in Kafka:

```bash
docker-compose exec kafka kafka-console-consumer --bootstrap-server kafka:29092 --topic twitch-streams --from-beginning
```

## Materialize

Connect to Materialize using the `mzcli`:

```bash
docker-compose run mzcli
```

### Create Sources

#### Kafka JSON source

The first step to consume our JSON events in Materialize is to create a [Kafka+JSON source](https://materialize.com/docs/sql/create-source/json-kafka/):

```sql
CREATE SOURCE kafka_twitch
FROM KAFKA BROKER 'kafka:9092' TOPIC 'twitch-streams'
  KEY FORMAT BYTES
  VALUE FORMAT BYTES
ENVELOPE UPSERT;
```

Because the data is stored as raw bytes, we need to do some casting to convert it to a readable format:

```sql
CREATE VIEW v_twitch_stream_conv AS
SELECT CAST(data AS JSONB) AS data
FROM (
      SELECT CONVERT_FROM(data, 'utf8') AS data
      FROM kafka_twitch
);

CREATE VIEW v_twitch_stream AS
SELECT CAST(nullif((data->>'id')::string, '') AS bigint)AS id,
       CAST(nullif((data->>'user_id')::string, '') AS int) AS user_id,
       (data->>'user_login')::string AS user_login,
       (data->>'user_name')::string AS user_name,
       CAST(nullif((data->>'game_id')::string, '') AS int) AS game_id,
       (data->>'game_name')::string AS game_name,
       (data->>'type')::string AS type,
       (data->>'title')::string AS title,
       (data->>'viewer_count')::int AS viewer_count,
       (data->>'started_at')::timestamp AS started_at,
       (data->>'language')::string AS language,
       (data->>'thumbnail_url')::string AS thumbnail_url,
       LIST[(data->'tag_ids'->>0)::string,(data->'tag_ids'->>1)::string,(data->'tag_ids'->>2)::string,(data->'tag_ids'->>3)::string,(data->'tag_ids'->>4)::string] AS tag_ids,
       (data->>'is_mature')::boolean AS is_mature
FROM v_twitch_stream_conv;
```

#### Postgres source

```sql
CREATE MATERIALIZED SOURCE mz_source 
FROM POSTGRES
  CONNECTION 'host=postgres port=5432 user=postgres dbname=postgres password=postgres'
  PUBLICATION 'mz_source';
```

```sql
CREATE VIEWS FROM SOURCE mz_source (stream_tag_ids);
```

## Ask questions!

### What are the most popular games on Twitch?

```sql
CREATE MATERIALIZED VIEW mv_agg_stream_game AS
SELECT game_name,
       COUNT(id) AS cnt_streams,
       SUM(viewer_count) AS agg_viewer_cnt
FROM v_twitch_stream
WHERE game_id IS NOT NULL
GROUP BY game_name;
```

#### What are the top10 games being played?

```sql
SELECT game_name, agg_viewer_cnt FROM mv_agg_stream_game ORDER BY agg_viewer_cnt DESC LIMIT 10;
```

#### Is anyone playing DOOM?

```sql
SELECT * FROM mv_agg_stream_game WHERE upper(game_name) LIKE 'DOOM%';
```

### What streams started in the last 15 minutes?

```sql
CREATE MATERIALIZED VIEW mv_stream_15min AS
SELECT title,
       user_name,
       game_name,
     started_at
FROM v_twitch_stream
WHERE game_id IS NOT NULL
  AND (mz_logical_timestamp() >= (extract('epoch' from started_at)*1000)::bigint 
  AND mz_logical_timestamp() < (extract('epoch' from started_at)*1000)::bigint + 900000);
```

```sql
SELECT MIN(started_at) FROM mv_stream_15min;
```

### What are the most popular tags?

```sql
CREATE MATERIALIZED VIEW mv_agg_stream_tag AS
SELECT st.localization_name AS tag,
       cnt_tag
FROM (
       SELECT tg, COUNT(*) AS cnt_tag
       FROM v_twitch_stream ts,
            unnest(tag_ids) tg 
       WHERE game_id IS NOT NULL
       GROUP BY tg
     ) un
JOIN stream_tag_ids st ON un.tg = st.tag_id AND NOT st.is_auto;
```

```sql
SELECT * FROM mv_agg_stream_tag ORDER BY cnt_tag DESC;
```

### Who are the most popular streamers for each of the top10 games?

```sql
CREATE VIEW v_stream_game_top10 AS 
SELECT game_id, game_name, agg_viewer_cnt 
FROM mv_agg_stream_game 
ORDER BY agg_viewer_cnt DESC 
LIMIT 10;
```

```sql
CREATE MATERIALIZED VIEW mv_stream_game_top10 AS
SELECT t.game_name, user_name, sum_viewer_cnt
FROM v_stream_game_top10 t,
LATERAL (
          SELECT game_name, user_name, SUM(viewer_count) AS sum_viewer_cnt
          FROM v_twitch_stream ts
          WHERE t.game_id = ts.game_id
            AND game_id IS NOT NULL
          GROUP BY game_name, user_name
          ORDER BY sum_viewer_cnt DESC
          LIMIT 1
        );
```

## Metabase

To visualize the results, navigate to (http://localhost:3030).


There should be a default dashboard named "What's streaming on Twitch?" listed under `Dashboards`:

<hr>

