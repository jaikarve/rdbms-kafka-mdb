# rdbms-kafka-mdb
Demo of using Kafka to do ETL (RDBMS &lt;-> MDB)

## Setup

__1. Configure Laptop__

* Install prerequisites: Docker, psql (`brew install postgresql`)
* Download and run Postgres docker image
  * Pull down the postgres docker image (`docker pull postgres`)
  * Create a directory to serve as localhost mount point for Postgres data: `mkdir -p $HOME/docker`
  * Startup Postgres docker container: ```docker run --rm   --name pg-docker -e POSTGRES_PASSWORD=docker -d -p 5432:5432 -v $HOME/docker/volumes/postgres:/var/lib/postgresql/data  postgres```
  * Start up the psql client: `psql -h localhost -U postgres -d postgres`
  * Run the script, `populate_table.sh`
  
* Download and run Confluent docker image
  * Start Confluent container
    * cd cp-all-in-one/
    * docker-compose up -d --build
  * Create JDBC Source Connector
    * ```curl --location --request POST 'localhost:8083/connectors' \
--header 'Content-Type: application/json' \
--data-raw '{
        "name": "cityweather-source",
        "config": {
                "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
                "connection.url": "jdbc:postgresql://pg-docker:5432/postgres",
                "connection.user": "postgres",
                "connection.password": "docker",
                "topic.prefix": "postgres-",
                "mode":"incrementing",
                "incrementing.column.name": "id",
                "name":"cityweather-source"
        }
}'```

* Start an interactive shell with the schema-registry container with the following command: `docker exec -it schema-registry /bin/bash`
* Start the AVRO console consumer with the following command: `./kafka-avro-console-consumer --topic postgres-citydata --bootstrap-server broker:29092 --from-beginning`
* Run the following command to see the messages in the topic: `docker exec -it schema-registry /usr/bin/kafka-avro-console-consumer --topic postgres-citydata --bootstrap-server broker:29092 --from-beginning`
* To join the city data to the weather data, we will create a Table in KSQL on the Weather topic and a Table in KSQL on the City Data Topic
  * Start KSQL: `docker-compose exec ksql-cli ksql http://ksql-server:8088`
  * Create stream based on topic: `CREATE STREAM POSTGRES_CITYDATA_SRC WITH (KAFKA_TOPIC='postgres-citydata', VALUE_FORMAT='AVRO');`
  * Set the topic offset to `earliest` so we have all of the data to date: `SET 'auto.offset.reset' = 'earliest';`
  * Create a stream with the citydata rekeyed by the city name, since that will be the field to join the tables on: `CREATE STREAM CITYDATA_REKEYED AS SELECT * FROM POSTGRES_CITYDATA_SRC PARTITION BY CITYNAME;`
  * Ensure that the CITYDATA_REKEYED stream is appropriately configured: `DESCRIBE EXTENDED CITYDATA_REKEYED;`
  * Create the CityData table: `CREATE TABLE CITYDATA WITH (KAFKA_TOPIC='CITYDATA_REKEYED', VALUE_FORMAT='AVRO', KEY='CITYNAME');`
  * Check that the CityData table contains the records: `SELECT * FROM CITYDATA EMIT CHANGES;`
  * Create a stream based on topic: `CREATE STREAM WEATHERDATA_SRC WITTH (KAFKA_TOPIC='postgres-weatherdata', VALUE_FORMAT='AVRO');`
  * Create a stream with the weatherdata rekeyed by the city name: `CREATE STREAM WEATHERDATA_REKEYED AS SELECT * FROM WEATHERDATA_SRC PARTITION BY CITYNAME;`
  * Create the WeatherData table: `CREATE TABLE WEATHERDATA WITH (KAFKA_TOPIC='WEATHERDATA_REKEYED', VALUE_FORMAT='AVRO', KEY='CITYNAME');`
  * SQL query for KSQL materialized view: `CREATE TABLE CITYWEATHERDATA AS SELECT C.CITYNAME, C.POPULATION, W.TEMP_HI, W.TEMP_LO FROM CITYDATA C INNER JOIN WEATHERDATA W ON C.CITYNAME = W.CITYNAME;`

* To create the MongoDB Sink Connector, run the following:
  * ```curl --location --request POST 'http://localhost:8083/connectors' \
--header 'Content-Type: application/json' \
--data-raw '{
	"name":"cityweather-sink",
	"config":{
		"connector.class":"com.mongodb.kafka.connect.MongoSinkConnector",
		"topics": "CITYWEATHERDATA",
		"connection.uri": "mongodb+srv://main_user:Nightmare69@fulltextsearch-nj9sp.mongodb.net/test",
		"database": "kafka-postgres",
		"collection": "city_weather_data",
		"document.id.strategy":"com.mongodb.kafka.connect.sink.processor.id.strategy.BsonOidStrategy",
		"key.converter":"org.apache.kafka.connect.storage.StringConverter",
		"value.converter":"io.confluent.connect.avro.AvroConverter",
		"value.converter.schema.registry.url":"http://schema-registry:8081",
		"transforms":"WrapKey",
		"transforms.WrapKey.type":"org.apache.kafka.connect.transforms.HoistField$Key",
		"transforms.WrapKey.field":"_id"
	}
}'```


# Delete Kafka Topic
  `./kafka-topics --topic orcl-GET_MONGODB_DOC --delete --zookeeper zookeeper:2181`
  
# Retrieving data from kafka topic

  `docker exec -it <name_of_broker_instance> /bin/bash`
  `./kafka-console-consumer.sh --topic test1-GET_MONGODB_DOC --bootstrap-server expandjsonsmt_kafka_1:9092 --from-beginning`




