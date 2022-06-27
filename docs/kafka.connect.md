A working (test) docker-compose (brain dump of the first encounter)



```
---
version: '3'
services:
  lvtn_postgres:
    image: postgres:14.1-alpine
    restart: always
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=test
    ports:
      - '25432:5432'
    volumes:
      - .tmp/postgres:/var/lib/postgresql/data

  kafka-ui:
    container_name: kafka-ui
    image: provectuslabs/kafka-ui:latest
    ports:
      - 8080:8080
    depends_on:
      - zookeeper0
      - zookeeper1
      - kafka0
      - kafka1
      - schemaregistry0
      - kafka-connect0
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka0:29092
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper0:2181
      KAFKA_CLUSTERS_0_JMXPORT: 9997
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schemaregistry0:8085
      KAFKA_CLUSTERS_0_KAFKACONNECT_0_NAME: first
      KAFKA_CLUSTERS_0_KAFKACONNECT_0_ADDRESS: http://kafka-connect0:8083
      KAFKA_CLUSTERS_1_NAME: secondLocal
      KAFKA_CLUSTERS_1_BOOTSTRAPSERVERS: kafka1:29092
      KAFKA_CLUSTERS_1_ZOOKEEPER: zookeeper1:2181
      KAFKA_CLUSTERS_1_JMXPORT: 9998
      KAFKA_CLUSTERS_1_SCHEMAREGISTRY: http://schemaregistry1:8085
      KAFKA_CLUSTERS_1_KAFKACONNECT_0_NAME: first
      KAFKA_CLUSTERS_1_KAFKACONNECT_0_ADDRESS: http://kafka-connect0:8083

  zookeeper0:
    image: confluentinc/cp-zookeeper:7.0.1
    container_name: lvtn-zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    healthcheck:
      test: echo srvr | nc lvtn_zookeeper 2181 || exit 1
      retries: 3
      interval: 10s

  kafka0:
    image: confluentinc/cp-kafka:7.0.1
    depends_on:
      - zookeeper0
    ports:
      - 9092:9092
      - 9997:9997
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper0:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka0:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      JMX_PORT: 9997
      KAFKA_JMX_OPTS: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.rmi.server.hostname=kafka0 -Dcom.sun.management.jmxremote.rmi.port=9997
    volumes:
      - .tmp/kafka0:/var/lib/kafka/data

  zookeeper1:
    image: confluentinc/cp-zookeeper:7.0.1
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka1:
    image: confluentinc/cp-kafka:7.0.1
    depends_on:
      - zookeeper1
    ports:
      - 9093:9093
      - 9998:9998
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper1:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka1:29092,PLAINTEXT_HOST://localhost:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      JMX_PORT: 9998
      KAFKA_JMX_OPTS: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Djava.rmi.server.hostname=kafka1 -Dcom.sun.management.jmxremote.rmi.port=9998
    volumes:
      - .tmp/kafka1:/var/lib/kafka/data

  schemaregistry0:
    image: confluentinc/cp-schema-registry:7.1.0
    ports:
      - 8085:8085
    depends_on:
      - zookeeper0
      - kafka0
    environment:
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: PLAINTEXT://kafka0:29092
      SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL: zookeeper0:2181
      SCHEMA_REGISTRY_KAFKASTORE_SECURITY_PROTOCOL: PLAINTEXT
      SCHEMA_REGISTRY_HOST_NAME: schemaregistry0
      SCHEMA_REGISTRY_LISTENERS: http://schemaregistry0:8085

      SCHEMA_REGISTRY_SCHEMA_REGISTRY_INTER_INSTANCE_PROTOCOL: "http"
      SCHEMA_REGISTRY_LOG4J_ROOT_LOGLEVEL: INFO
      SCHEMA_REGISTRY_KAFKASTORE_TOPIC: _schemas

  schemaregistry1:
    image: confluentinc/cp-schema-registry:7.1.0
    ports:
      - 18085:8085
    depends_on:
      - zookeeper1
      - kafka1
    environment:
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: PLAINTEXT://kafka1:29092
      SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL: zookeeper1:2181
      SCHEMA_REGISTRY_KAFKASTORE_SECURITY_PROTOCOL: PLAINTEXT
      SCHEMA_REGISTRY_HOST_NAME: schemaregistry1
      SCHEMA_REGISTRY_LISTENERS: http://schemaregistry1:8085

      SCHEMA_REGISTRY_SCHEMA_REGISTRY_INTER_INSTANCE_PROTOCOL: "http"
      SCHEMA_REGISTRY_LOG4J_ROOT_LOGLEVEL: INFO
      SCHEMA_REGISTRY_KAFKASTORE_TOPIC: _schemas

  kafka-connect0:
    image: lvtn_connect
    build:
      context: .
      dockerfile: ./Dockerfile.kafka-connect
    ports:
      - 8083:8083
    depends_on:
      - kafka0
      - schemaregistry0
    environment:
      CONNECT_BOOTSTRAP_SERVERS: kafka0:29092
      CONNECT_GROUP_ID: compose-connect-group
      CONNECT_CONFIG_STORAGE_TOPIC: _connect_configs
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_OFFSET_STORAGE_TOPIC: _connect_offset
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_STATUS_STORAGE_TOPIC: _connect_status
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.storage.StringConverter
      CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL: http://schemaregistry0:8085
      CONNECT_VALUE_CONVERTER: org.apache.kafka.connect.storage.StringConverter
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schemaregistry0:8085
      CONNECT_INTERNAL_KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_INTERNAL_VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      CONNECT_REST_ADVERTISED_HOST_NAME: kafka-connect0
      CONNECT_PLUGIN_PATH: "/usr/share/java,/usr/share/confluent-hub-components"

  kafka-init-topics:
    image: confluentinc/cp-kafka:7.0.1
    volumes:
      - ./message.json:/data/message.json
    depends_on:
      - kafka1
    command: "bash -c 'echo Waiting for Kafka to be ready... && cub kafka-ready -b kafka1:29092 1 30 && kafka-topics --create --topic second.users --partitions 3 --replication-factor 1 --if-not-exists --zookeeper zookeeper1:2181 && kafka-topics --create --topic second.messages --partitions 2 --replication-factor 1 --if-not-exists --zookeeper zookeeper1:2181 && kafka-topics --create --topic first.messages --partitions 2 --replication-factor 1 --if-not-exists --zookeeper zookeeper0:2181 && kafka-console-producer --broker-list kafka1:29092 -topic second.users < /data/message.json'"
```

The PSQL has to have WAL activated

I do it manually by editing `.tmp/postgresql.conf`

```
wal = logical_replication
```

After that, the following config needs to written to Kafka Connect:

```
{
  "name": "test-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "lvtn_postgres",
    "database.port": "25432",
    "database.user": "postgres",
    "database.password": "postgres",
    "database.dbname" : "test",
    "database.server.name": "test",
    "table.include.list": "public.fruits",
    "plugin.name" : "pgoutput"

  }
}


[appuser@31ea9ab50187 ~]$ curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{"name": "test-connector", "config": {"connector.class": "io.debezium.connector.postgresql.PostgresConnector", "database.hostname": "lvtn_postgres", "database.port": "5432", "database.user": "postgres", "database.password": "postgres", "database.dbname": "test", "database.server.name": "test", "table.include.list": "public.fruits"}}'


```


Note, the port is local to the docker network - so inside containers, there is no masking. Also, the plugin.name has to be `pgoutput` otherwise it defaults to `decoderbufs`

The log output of `kafka-connect` container is useful to tail.

And I didn't need to give replication privileges (probably because the `postgres` user is a superuser)

After this config, a new topic `test.public.fruits` appears - and I verified it works by (painstaikingly) inserting data into a postgres table. The stream of events then appears in Kafka, eg.

```
Struct{after=Struct{id=1,fingerprint=foofruit,url=http://google.com},source=Struct{version=1.9.2.Final,connector=postgresql,name=test,ts_ms=1656362754188,db=test,sequence=["24366712","24367104"],schema=public,table=fruits,txId=740,lsn=24367104},op=c,ts_ms=1656362754283}

Struct{after=Struct{id=1,fingerprint=foofruit,url=http://google.com/foo},source=Struct{version=1.9.2.Final,connector=postgresql,name=test,ts_ms=1656362932704,db=test,sequence=["24367568","24367624"],schema=public,table=fruits,txId=741,lsn=24367624},op=u,ts_ms=1656362932757}
```

Corresponding to:

```
insert into fruits (fingerprint, url) values ('foofruit', 'http://google.com');

update fruits SET url='http://google.com/foo' where fingerprint='foofruit';

```

A tiny victory after whole day of work, many problems:

- docker-compose reusing data from previous containers
    - do `docker-compose down`
    - use `--remove-orphans`
    - read logs! (lots of output)
    - debug each container individually
- i had no idea what config works
    - assembled docker-compose multiple times
    - got inspiration from kafka-ui
- btw, i'm running out of RAM...
