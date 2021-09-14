# What's streaming on Twitch?

In this demo, we'll try to make some sense of Twitch using **[Materialize](https://materialize.com/docs/)** and some good ol' **standard SQL**.

**First things first** :point_down:

To work with data from Twitch, you need to [register an app](https://dev.twitch.tv/docs/authentication#registration) and get a hold of your app access tokens. If you already have an account, the process should be pretty smooth! After cloning this repo, remember to replace `<client_id>` and `<client_secret>` in the [Kafka producer file](./data-generator/twitch_kafka_producer.py) with the valid credentials.

## Docker

We'll use Docker Compose to make it easier to bundle up all the services for our Twitch analytics pipeline:

<p align="center">
<img width="650" alt="demo_overview" src="https://user-images.githubusercontent.com/23521087/133231555-07a92479-de4d-44b1-b0c9-6fa819e3cbcb.png">
</p>

#### Getting the setup up and running

```bash

# Start the setup
docker-compose up -d

# Is everything really up and running?
docker-compose ps
```

## Kafka

The data generator produces **JSON-formatted** events about [active streams](https://dev.twitch.tv/docs/api/reference/#get-streams) on Twitch into the `twitch-streams` Kafka topic. To check that the topic has been created:

```bash
docker-compose exec kafka kafka-topics \
    --list \
    --bootstrap-server kafka:9092
```

and that there's data landing:

```bash
docker-compose exec kafka kafka-console-consumer \
    --bootstrap-server kafka:9092 \
    --topic twitch-streams \
    --from-beginning
```

## Postgres

Postgres is bootstrapped with the list of [stream tags](https://dev.twitch.tv/docs/api/reference/#get-all-stream-tags) that can be used to describe live streams and categories. This reference data allows us to enrich `twitch-streams` events with the description for each `tag_id`.

## Materialize

To connect to the running Materialize service, we can use any [compatible CLI](https://materialize.com/docs/connect/cli/). Because we're on Docker, let's roll with `mzcli`:

```bash
docker-compose run mzcli
```

### Create Sources

#### Kafka JSON source

**1.** The first step to consume JSON events from Kafka in Materialize is to create a [Kafka+JSON source](https://materialize.com/docs/sql/create-source/json-kafka/):

```sql
CREATE SOURCE kafka_twitch
FROM KAFKA BROKER 'kafka:9092' TOPIC 'twitch-streams'
  KEY FORMAT BYTES
  VALUE FORMAT BYTES
ENVELOPE UPSERT;
```

>**Why do we need `ENVELOPE UPSERT`?**
>
>The Twitch Helix API returns events sorted by number of current viewers. This means that there might be **duplicate** or **missing events** as viewers join and leave a broadcast — which we'll have to deal with. As new events flow in for the same Kafka key (`id`), we're only ever interested in keeping the most recent values, so we'll use [`ENVELOPE UPSERT`](https://materialize.com/docs/sql/create-source/json-kafka/#upsert-envelope-details) to make sure Materialize can also handle updates (and the associated retractions!).

**2.** The data is stored as raw bytes, so we need to do some casting to convert it to a readable format (and appropriate data types) next:

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

**1.** One way to connect to a Postgres database in Materialize is to use a [Postgres source](https://materialize.com/docs/sql/create-source/postgres/), which uses [logical replication](https://www.postgresql.org/docs/10/logical-replication.html) to continuously ingest changes and maintain freshly updated results:

```sql
CREATE MATERIALIZED SOURCE mz_source 
FROM POSTGRES
  CONNECTION 'host=postgres port=5432 user=postgres dbname=postgres password=postgres'
  PUBLICATION 'mz_source';

CREATE VIEWS FROM SOURCE mz_source (stream_tag_ids);
```
 
### Ask questions!

#### What are the most popular games on Twitch?

```sql
CREATE MATERIALIZED VIEW mv_agg_stream_game AS
SELECT game_id,
       game_name,
       COUNT(id) AS cnt_streams,
       SUM(viewer_count) AS agg_viewer_cnt
FROM v_twitch_stream
WHERE game_id IS NOT NULL
GROUP BY game_id, game_name;
```

**What are the top10 games being played?**

```sql
SELECT game_name, 
       cnt_streams, 
       agg_viewer_cnt 
FROM mv_agg_stream_game 
ORDER BY agg_viewer_cnt 
DESC LIMIT 10;
```

**Is anyone playing DOOM?**

```sql
SELECT game_name, 
       cnt_streams, 
       agg_viewer_cnt 
FROM mv_agg_stream_game 
WHERE upper(game_name) LIKE 'DOOM%';
```

#### What gaming streams started in the last 15 minutes?

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

#### What are the most used tags?

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

#### Who are the most popular streamers for each of the top10 games?

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

To visualize the results in [Metabase](https://www.metabase.com/):

**1.** In a browser, navigate to <localhost:3030> (or <`host`:3030>, if running on a VM).

**2.** Click **Let's get started**.

**3.** Complete the first set of fields asking for your email address. This
    information isn't crucial for anything but has to be filled in.

**4.** On the **Add your data** page, specify the connection properties for the Materialize database:

Field             | Value
----------------- | ----------------
Database          | Materialize
Name              | twitch
Host              | **materialized**
Port              | **6875**
Database name     | **materialize**
Database username | **materialize**
Database password | Leave empty

**5.** Click **Ask a question** -> **Native query**.

**6.** Under **Select a database**, choose **twitch**.

**7.** In the query editor, enter:

```sql
SELECT SUM(cnt_streams) FROM mv_agg_stream_game;
```
    
and hit **Save**. You need to do this for each visualization you’re planning to add to the dashboard that Metabase prompts you to create.

**8.** Once you have a dashboard set up, you can manually set the refresh rate to 1 second by adding `#refresh=1` to the end of the URL: 

`http://localhost:3030/dashboard/1-whats-streaming-on-twitch#refresh=1`

and opening the modified URL in a new tab:

![Metabase](https://user-images.githubusercontent.com/23521087/130734450-9b5d2225-58ed-472f-96a2-976e4b72d1e9.gif)

<hr>

### TODO

- [ ] **Improve the Kafka producer:** 

    In the future, it'd be preferable to use [PubSub](https://dev.twitch.tv/docs/pubsub) to subscribe to a topic for updates instead of periodically sending requests to the Twitch Helix API.
    
- [ ] **Pre-load the Metabase dashboard:** 

    Include a backup of Metabase's [embedded database](https://www.metabase.com/docs/latest/operations-guide/backing-up-metabase-application-data.html) (H2) with a bootstrapped dashboard to save users some time in this step.
