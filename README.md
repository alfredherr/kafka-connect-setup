# Kafka Connect Set Up

## Prerequisites

 * Docker (Make sure it has at least 8GB allocated)
 * [Confluent Platform](https://www.confluent.io/download)
 * [Confluent CLI](https://docs.confluent.io/current/cli/installing.html#cli-install) (requires separate installation)

You should add both to your PATH, for example, in your `~/.bashrc`: 

```bash
export PATH="$PATH:/Users/alfredoherrera/work/pipefy/confluent/cli/bin"
export PATH="$PATH:/Users/alfredoherrera/work/pipefy/confluent/confluent-5.3.1/bin"
export  CONFLUENT_HOME="$HOME/work/pipefy/confluent/confluent-5.3.1"
```

## Running in Docker

```bash
docker-compose up
```

### Listing topics: 

kafka-topics --list --bootstrap-server localhost:9092

### Consuming (avro): 

kafka-avro-console-consumer --topic asgard.public.customers  \
  --bootstrap-server localhost:9092 \
  --property schema.registry.url=http://localhost:8081 \
  --property print.key=true \
  --key-deserializer=org.apache.kafka.common.serialization.StringDeserializer \
  --from-beginning

### Listing schemas: 

 - curl -s http://localhost:8081/subjects/

## Registering Source Connector

curl -i -X POST \
    -H "Accept:application/json" \
    -H  "Content-Type:application/json" \
    http://localhost:8083/connectors/ -d '
    {
        "name": "pg-source",
        "config": {
            "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
            "tasks.max": "1",
            "database.hostname": "postgres",
            "database.port": "5432",
            "database.user": "postgres",
            "database.password": "postgres",
            "database.dbname" : "postgres",
            "database.server.name": "asgard",
            "database.history.kafka.bootstrap.servers": "kafka:29092",
            "database.history.kafka.topic": "schema-changes.pg",
            "transforms": "unwrap,InsertTopic,InsertSourceDetails",
            "transforms.unwrap.type": "io.debezium.transforms.UnwrapFromEnvelope",
            "transforms.InsertTopic.type":"org.apache.kafka.connect.transforms.InsertField$Value",
            "transforms.InsertTopic.topic.field":"messagetopic",
            "transforms.InsertSourceDetails.type":"org.apache.kafka.connect.transforms.InsertField$Value",
            "transforms.InsertSourceDetails.static.field":"messagesource",
            "transforms.InsertSourceDetails.static.value":"Debezium CDC from Postgres on asgard"
        }
    }'

## Registering Sink Connector

curl -i -X POST \
    -H "Accept:application/json" \
    -H  "Content-Type:application/json" \
    http://localhost:18083/connectors/ -d '
{
   "name":"jdbc-sink",
   "config":{
        "connection.password": "postgres",
        "connection.url": "jdbc:postgresql://postgres:5432/postgres",
        "connection.user": "postgres",
        "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
        "errors.deadletterqueue.context.headers.enable": "true",
        "errors.deadletterqueue.topic.name": "jdbc.sink.deadletter",
        "errors.deadletterqueue.topic.replication.factor": "1",
        "errors.tolerance": "all",
        "fields.whitelist": "first_name,last_name,email",
        "insert.mode": "upsert",
        "key.converter": "org.apache.kafka.connect.storage.StringConverter",
        "metadata.max.age.ms": "30000",
        "pk.fields": "id",
        "pk.mode": "record_value",
        "table.name.format": "customer_emails",
        "topics": "asgard.public.customers",
        "value.converter": "io.confluent.connect.avro.AvroConverter",
        "value.converter.schema.registry.url": "http://schema-registry:8081"
   }
}'

### Checking connector status

 * https://docs.confluent.io/current/connect/references/restapi.html

curl -i -X GET -H "Accept:application/json" -H  "Content-Type:application/json"  http://localhost:18083/connectors/

## Other

Tutorial based on [this one](https://github.com/confluentinc/examples/tree/5.3.1-post/postgres-debezium-ksql-elasticsearch)
