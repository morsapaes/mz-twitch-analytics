FROM python:3.8-slim

RUN set -ex; \
  	pip install --no-cache-dir confluent-kafka; \
  	pip install --no-cache-dir python-twitch-client;

ADD twitch_kafka_producer.py /

CMD ["python", "-u", "./twitch_kafka_producer.py"]