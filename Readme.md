

In this project we are creating a data pipeline which will fetch the data from our postgres database using jdbc connector and then send data to kafka connect , then in kafka connect we will use s3 sink connector to send data to s3 bucket . This project is done using confluent kafka community version , which is free to use.

## ARCHITECTURE

![image](https://github.com/user-attachments/assets/999a3f59-a5c9-45be-9f26-e4d656eded7d)

## Prerequisites

- Docker and docker-compose
- aws cli

## Steps

1. [Install all the required confluent kafka component using confluent kafka documentation](https://docs.confluent.io/platform/current/get-started/platform-quickstart.html)

   ```
   wget https://raw.githubusercontent.com/confluentinc/cp-all-in-one/7.7.1-post/cp-all-in-one-kraft/docker-compose.yml
   ```
   ![image](https://github.com/user-attachments/assets/9e54c53f-30f9-4a0b-9bcc-5db57385f1b7)

   **NOTE** : I have installed postgres as well using docker compose as well that's why you are seeing a postgres container as well in the image .

2 . Go into `connect` container

    ```
    docker exec -it acda1215b1f7 /bin/bash
    ```

3. Install jdbc driver in kafka connect
   ```
   confluent-hub install confluentinc/kafka-connect-jdbc:latest
   ```

   - After this restart your container
     ```
     docker restart <container_id_or_container_name>
     ```

4. Now create a connector config file , keep it in mind that it is written in `json` format

   - I have created this in `/etc/kafka-connect/` location
   ```
   [appuser@connect kafka-connect]$ cat connector-config.json 
    {
      "name": "test-source-postgres",
      "config": {
        "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
        "tasks.max": "1",
        "connection.url": "jdbc:postgresql://postgres:5432/mydb",
        "connection.user": "postgres",
        "connection.password": "password",
        "mode": "incrementing",
        "incrementing.column.name": "id",
        "topic.prefix": "postgres-",
        "table.whitelist": "employees",
        "poll.interval.ms": "5000"
      }
    }
    ```

5. Now , let's create this connector
   ```
   curl -X POST -H "Content-Type: application/json" --data @/etc/kafka-connect/connector-config.json http://localhost:8083/connectors
   ```
   **NOTE:** if your postgres password and username matches then only kafka will create a topic and connector
   ![topicc](https://github.com/user-attachments/assets/98c3bd60-4f4a-491d-99f0-ad1dc503d92d)


   ![Real-time-data-in-kafka](https://github.com/user-attachments/assets/f0e39efe-b69b-429b-ad14-99a6af06453c)


6. Now , lets create a s3 sink connector , for this let's first install `s3 connector`

   ```
   confluent-hub install confluentinc/kafka-connect-s3:latest
   ```

   - After this restart your container
     ```
     docker restart <container_id_or_container_name>
     ```
     
7. let's create s3 configuration for s3 connector

   ```
    echo '
    {
      "name": "s3-sink-connector",
      "config": {
        "connector.class": "io.confluent.connect.s3.S3SinkConnector",
        "tasks.max": "1",
        "topics": "postgres-employees",
        "s3.bucket.name": "postgres-kafka",
        "s3.region": "ap-south-1",
        "s3.part.size": "5242880",
        "flush.size": "3",
        "storage.class": "io.confluent.connect.s3.storage.S3Storage",
        "format.class": "io.confluent.connect.s3.format.avro.AvroFormat",
        "schema.compatibility": "NONE",
        "key.converter": "org.apache.kafka.connect.storage.StringConverter",
        "value.converter": "io.confluent.connect.avro.AvroConverter",
        "value.converter.schema.registry.url": "http://schema-registry:8081",
        "partitioner.class": "io.confluent.connect.storage.partitioner.DefaultPartitioner",
        "aws.access.key.id": "<your_access_key>",
        "aws.secret.access.key": "<your_secret_key>"
   
      }
    } ' > s3-sink-connector.json

    ```

   NOTE: if you don't want to put your access key and secret key in s3 configuration json file , then you should attach `s3 access` iam role to your instance.
   
   - Now do
       ```
       curl -X POST -H "Content-Type: application/json" --data @/etc/kafka-connect/s3-sink-connector.json http://localhost:8083/connectors
       ```

   - if your whole configuration is right , then kafka will be able to connect with s3 and push data into it
   ![Screenshot 2024-09-25 012839](https://github.com/user-attachments/assets/6e065ad5-51b6-40df-873e-8757c8400053)

   

# SOME EXTRA DEBUGGING COMMANDS

- to see all the connectors
  ```
  curl http://localhost:8083/connectors
  ```

- to delete the created connector

  ```
  curl -X DELETE http://localhost:8083/connectors/<connector_name>
  ```
  
- To check the status of connector
  ```
  curl http://3.110.62.217:8083/connectors/<connector_name>/status
  ```



   
   
