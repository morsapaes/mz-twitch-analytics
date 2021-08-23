# What's streaming on Twitch?

## Docker

#### Getting the setup up and running
`docker-compose build`

`docker-compose up -d`

#### Is everything really up and running?

`docker-compose ps`

## Kafka

Check that the `twitch-streams` topic has been created:

`docker-compose exec kafka kafka-topics --list --bootstrap-server kafka:9092`

And that there's data landing in Kafka:

`docker-compose exec kafka kafka-console-consumer --bootstrap-server kafka:29092 --topic twitch-streams --from-beginning`

## Materialize

Connect to Materialize using the `mzcli`:

`docker-compose run mzcli`

#### Kafka JSON source

The first step to consume our JSON events in Materialize is to create a [Kafka+JSON source](https://materialize.com/docs/sql/create-source/json-kafka/):

```sql
CREATE MATERIALIZED SOURCE kafka_twitch
FROM KAFKA BROKER 'kafka:9092' TOPIC 'twitch-streams'
KEY FORMAT BYTES
VALUE FORMAT BYTES
ENVELOPE UPSERT;
```

Because the data is stored as raw bytes, we need to do some casting to convert it to a readable format:

```sql
CREATE VIEW v_twitch_stream AS
SELECT CAST(data AS JSONB) AS data
FROM (
    SELECT CONVERT_FROM(data, 'utf8') AS data
    FROM kafka_twitch
);
```

#### Postgres source

### Metabase

<hr>

