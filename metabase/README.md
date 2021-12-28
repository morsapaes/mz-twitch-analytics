# Metabase

To reproduce the dashboard in this repo:

`Ask a Question -> Native Query + Select a database -> Twitch`

You'll need to do this for each chart in the dashboard.

### Total Streams

```sql
SELECT SUM(cnt_streams) FROM mv_agg_stream_game;
```

**Save** -> Name: “Total Streams”

### Total Started (15 minutes)

```sql
SELECT COUNT(*) FROM mv_stream_15min;
```

**Save** -> Name: “Total Started (last 15m)”

>**Re-running the data generator**
>
> Because the data generator doesn't repeatedly call the Twitch API (by default), whenever data “falls out” of the 15 minute window _completely_, you can either manually trigger the generator using `docker-compose run -d data-generator` or enable [Ofelia](../data-generator/README.md#scheduling) to run it periodically.

### top10 Games

```sql
SELECT game_name,
       cnt_streams,
       agg_viewer_cnt
FROM mv_agg_stream_game
ORDER BY agg_viewer_cnt
DESC LIMIT 10;
```

**Save** -> Name: “Top 10 Games”

### Star Streamers (top10 Games)

```sql
SELECT game_name, user_name
FROM mv_stream_game_top10
ORDER BY sum_viewer_cnt DESC;
```

**Save** -> Name: “Star Streamers (Top10 Games)”

### Most Used Tags

```sql
SELECT * FROM mv_agg_stream_tag ORDER BY cnt_tag DESC;
```

**Save** -> Name: “Most Used Tags”
