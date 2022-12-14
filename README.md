# Data flow from Kafka Mirror to Web UI

## Goal

We are on the headquarter cluster. We have multiple kafka topics comming from multiple warehouses cluster (thanks to the mirror).  
We are going to create a Camel K Inegration to stream the different kafka topics and update a relationnal database recording the packet location.  
In parallel, we set up a Change Data Capture Kafka Connector (Debezium) on this database to capture any change. The changes are written into another Kafka topic.  
Thus, by configuring the Web UI application to listen to this topic, we will have information (at least) concerning packet id, packet last_postion, packet new_position.

## Deploy MySQL Database

0. Create a mysql namespace
1. Create a mysql database
```
oc new-app mysql-ephemeral -e MYSQL_ROOT_PASSWORD=rootpass
```
2. Run a shell in the newly created mysql pod and create a new table "track". This table will record the current packets location.
```
$ oc rsh -n mysql <MY_SQL_POD>
sh-4.4$ mysql -u root
mysql> use sampledb;
mysql> CREATE TABLE track (id varchar(255), warehouse varchar(255), PRIMARY KEY (id));
```

## Deploy Kafka, KafkaConnect and KafkaConnector

0. Create a namespace "streams"
1. Install the Red Hat Integration - AMQ Streams operator
2. Create a Kafka resource
```
kind: Kafka
apiVersion: kafka.strimzi.io/v1beta2
metadata:
  name: my-cluster
  namespace: streams
spec:
  kafka:
    version: 3.2.3
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: '3.2'
    storage:
      type: ephemeral
  zookeeper:
    replicas: 3
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {}

```
3. Wait for the status condition Ready
4. Before creating the KafkaConnect Resource, create an image stream:
```
$ oc create -n streams is debezium-streams-connect
```
5. Create a KafkaConnect Resource
```
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  annotations:
    strimzi.io/use-connector-resources: 'true'
  name: kafka-connect
  namespace: streams
spec:
  bootstrapServers: 'my-cluster-kafka-bootstrap.streams.svc.cluster.local:9092'
  build:
    output:
      image: 'debezium-streams-connect:latest'
      type: imagestream
    plugins:
      - artifacts:
          - type: zip
            url: >-
              https://maven.repository.redhat.com/ga/io/debezium/debezium-connector-mysql/1.9.7.Final-redhat-00003/debezium-connector-mysql-1.9.7.Final-redhat-00003-plugin.zip
        name: debezium-connector-mysql
  config:
    status.storage.topic: connect-cluster-status
    status.storage.replication.factor: -1
    offset.storage.topic: connect-cluster-offsets
    topic.creation.default.partitions: 10
    group.id: connect-cluster
    topic.creation.default.replication.factor: 3
    config.storage.replication.factor: -1
    config.storage.topic: connect-cluster-configs
    topic.creation.enable: 'true'
    topic.creation.default.cleanup.policy: compact
    topic.creation.default.compression.type: lz4
    offset.storage.replication.factor: -1
  replicas: 1
  version: 3.2.3
```
6. Wait for the status condition to be Ready
7. Create a KafkaConnector Resource
```
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  labels:
    strimzi.io/cluster: kafka-connect
  name: connector-mysql
  namespace: streams
spec:
  class: io.debezium.connector.mysql.MySqlConnector
  config:
    database.server.name: connector_mysql
    database.hostname: mysql.mysql.svc.cluster.local
    database.allowPublicKeyRetrieval: true
    database.password: rootpass
    database.port: 3306
    database.history.kafka.topic: schema-changes.history
    database.history.kafka.bootstrap.servers: 'my-cluster-kafka-bootstrap.streams.svc.cluster.local:9092'
    database.dbname: sampledb
    database.user: root
  tasksMax: 1
```
8. For this example, let's create 2 topics to simulate the mirrored topics from the warehouses clusters.
```
kind: KafkaTopic
apiVersion: kafka.strimzi.io/v1beta2
metadata:
  name: warehouse1
  labels:
    strimzi.io/cluster: my-cluster
  namespace: streams
spec:
  partitions: 10
  replicas: 3
  config:
    retention.ms: 604800000
    segment.bytes: 1073741824

```
```
kind: KafkaTopic
apiVersion: kafka.strimzi.io/v1beta2
metadata:
  name: warehouse2
  labels:
    strimzi.io/cluster: my-cluster
  namespace: streams
spec:
  partitions: 10
  replicas: 3
  config:
    retention.ms: 604800000
    segment.bytes: 1073741824

```
## Create a Camel K integration

0. Create a namespace "kamel"
1. Install the operator: Red Hat Integration - Camel K
2. Create the IntegrationPlatform Resource:
```
kind: IntegrationPlatform
apiVersion: camel.apache.org/v1
metadata:
  labels:
    app: camel-k
  name: camel-k
  namespace: kamel
spec:
  build: {}
  kamelet: {}
  profile: OpenShift
  resources: {}
status:
  build:
    maven:
      settings: {}
    registry: {}
  kamelet: {}
  resources: {}

```
3. Download the CLI from https://mirror.openshift.com/pub/openshift-v4/clients/camel-k/1.8.2/camel-k-client-1.8.2-linux-64bit.tar.gz (WARNING: Better to check link from openshift https://<CONSOLE_URL>/command-line-tools )
4. Create a JDBCInsert.java file contening:
```
package datasourceAutowired;

// camel-k: dependency=camel:jdbc
// camel-k: dependency=mvn:io.quarkus:quarkus-jdbc-mysql

import org.apache.camel.builder.RouteBuilder;

public class JDBCInsert extends RouteBuilder {

    @Override
    public void configure() throws Exception {
        from("kafka:warehouse1,warehouse2?brokers=my-cluster-kafka-bootstrap.streams.svc.cluster.local:9092")
                .setBody(simple("INSERT INTO track VALUES ('${body}','${headers[kafka.TOPIC]}') ON DUPLICATE KEY UPDATE warehouse='${headers[kafka.TOPIC]}';"))
                .to("jdbc:default")
                .to("log:info");
    }
}

```
5. Create the datasource.properties file
```
quarkus.datasource.jdbc.url=jdbc:mysql://mysql.mysql.svc.cluster.local:3306/sampledb
quarkus.datasource.username=root
quarkus.datasource.password=rootpass
```
6. Create a secret
```
oc create secret generic my-datasource --from-file=datasource.properties
```
7. Run the inegration
```
kamel run JDBCInsert.java --config secret:my-datasource
```
8. Wait until the Integration finish to build and deploy the camel k pod
## Test integration

1. Run a shell in a kafka pod
```
$ oc rsh -n streams my-cluster-kafka-0
sh-4.4$ ./bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic warehouse1
```
You can now produce kafka events to simulate a packet scanned in warehouse1.
```
>device1
>device2
```
You can also produce events in warehouse2 topics.
```
sh-4.4$ ./bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic warehouse1
>device1
>device2
```

2. Run a shell in the mysql pod as above and query the database while producing events
```
mysql> SELECT * FROM track;
```

3. Check in the Kafka Topics the topic created by Change Data Capture. A topic nammed *connector_mysql.sampledb.track* has been created. You can connect to a kafka pod and consume from the beginning the topic. This retrieves the changes in a JSON format with a "before" and "after" state.
```
$ oc rsh -n streams my-cluster-kafka-0
sh-4.4$ ./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic connector_mysql.sampledb.track --from-beginning
```
4. (Optionnal) Analyse log of the camel integration in kamel namespace
```
$ oc logs -n kamel <CAMEL_K_POD> -f
```
## Future

- We need to secure (tls + auth ?), add persistent storage, eventually scale ?
- We need to think about a way for the Web UI to listen to the Kafka topic from the Data Change Caputre component.