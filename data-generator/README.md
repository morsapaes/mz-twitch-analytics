# Twitch Data Generator

## 

To ingest Twitch data into Kafka, we’ll use a short script to


use existing Python wrappers for the Twitch Helix API ([`python-twitch-client`](https://github.com/tsifrer/python-twitch-client)) and Kakfa ([`confluent-kafka`](https://github.com/confluentinc/confluent-kafka-python)) to write a minimal producer (`data-generator/twitch_kafka_producer.py`).

## Producing data to Kafka

#### Twitch Authentication Flow

To work with data from Twitch, we first need to [register an app](https://dev.twitch.tv/docs/authentication#registration) and get a hold of our [app access tokens](). In this demo, we'll use the `streams` endpoint to produce events about active streams into Kafka, and the `/tags/streams` endpoint to grab all stream tags defined by Twitch.

<details>
<summary>Twitch Developer Console</summary>

![Kafka Producer App](https://user-images.githubusercontent.com/23521087/123405059-bfdc3080-d5a9-11eb-8976-473806f9097d.png)
</details> 

#### Kafka Producer (Python)



Two things to keep in mind about the data:

1. **Duplicates:** streams are returned sorted by number of current viewers, in descending order. Across multiple pages of results, there may be duplicate or missing streams, as viewers join and leave.

2. **Streams with multiple categories:** each stream may have up to five tags, so we'll end up with a JSON array literal for each record.

Check that the `twitch-streams` topic has been created:

`docker-compose exec kafka kafka-topics --list --bootstrap-server kafka:9092`

And that there's data landing in Kafka:

`docker-compose exec kafka kafka-console-consumer --bootstrap-server kafka:29092 --topic twitch-streams --from-beginning`

https://user-images.githubusercontent.com/23521087/123432486-bd3e0300-d5ca-11eb-80d2-5086396f0a55.mp4