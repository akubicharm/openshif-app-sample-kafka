# Kafka App 

![](./docs/app-architecture.png)


## Kafka (AMQ Stream のデプロイ)

* OperatorHubからAMQ Streamを選んでインストール
* インストール済みオペレーターから AMQ Streamsを選んで Kafka インスタンスを作成
* listener1 を追加

```
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
  namespace: amq-streams-kafka
spec:
  kafka:
    # ...
    listeners:
      # ...
      - name: listener1
        port: 9094
        type: route
        tls: true
# ...
```

## Processor/Producer アプリ

### 初期化処理
mvn io.quarkus.platform:quarkus-maven-plugin:3.9.4:create \
    -DprojectGroupId=org.acme \
    -DprojectArtifactId=kafka-quickstart-processor \
    -Dextensions='rest,openshift'

既存プロジェクトにOpenShift用のDependencyを追加する場合、
`./mvnw quarkus:add-extension -Dextensions="openshift"`
または `quarkus extension add openshift` を実行すると、pom.xml に以下のdependencyが追加される。
```
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-openshift</artifactId>
        </dependency> 
```



### アプリケーション実行時に必要な環境変数

`KAKFA_BOOTSTRAP_SERVERS` 環境変数の設定が必要。
kafkaを my-clusterという名前で追加した場合は以下の通り。
```
KAFKA_BOOTSTRAP_SERVERS: my-cluster-kafka-bootstrap:9092
```

### OpenShiftへのデプロイ (Git ストラテジー)

BuildStrategy = git でs2i ビルドを利用する場合には、Runnable Jarを生成する必要があるので、pom.xml に `<quarkus.package.true>uber-jar</quarkus.package.type>`　が必要。

```
    <properties>
        <quarkus.package.type>uber-jar</quarkus.package.type>
    </properties>
```

### OpenShiftへのデプロイ　(mvn)
```
mvn install -Dquarkus.openshift.deploy=true -Dquarkus.openshift.route.expose=true -Dquarkus.openshift.env.vars.kafka-bootstrap-servers=my-cluster-kafka-bootstrap:9092
```

`-Dquarkus.openshift.env.vars.kafka-bootstrap-servers=my-cluster-kafka-bootstrap:9092` をパラメータに指定するとコンテナにKAFKA_BOOTSTRAP_SERVERS=my-cluster-kafka-bootstrap:9092 という環境変数が設定される。



## processor　アプリ固有の設定

スケールした時にoffsetを共有できるように consumer group を application.propertiesに設定しておく。
ここで指定したconsumer groupの名前をscaledObjectにも設定する。

```
%dev.quarkus.http.port=8081

# Go bad to the first records, if it's out first access
kafka.auto.offset.reset=earliest

# Set the Kafka topic, as it's not the channel name
mp.messaging.incoming.requests.topic=quote-requests


# Configure the outgoing `quotes` Kafka topic
mp.messaging.outgoing.quotes.value.serializer=io.quarkus.kafka.client.serialization.ObjectMapperSerializer

mp.messaging.incoming.requests.group.id=mygroup
```


## producerアプリ固有の設定
特になし


## Topic 送受信テスト

```
oc run kafka-consumer -ti \
--image=registry.redhat.io/amq-streams/kafka-36-rhel8:2.6.0 \
--rm=true \
--restart=Never \
-- bin/kafka-console-consumer.sh \
--bootstrap-server my-cluster-kafka-bootstrap:9092 \
--topic quote-requests \
--from-beginning


oc run kafka-producer -ti \
--image=registry.redhat.io/amq-streams/kafka-36-rhel8:2.6.0 \
--rm=true \
--restart=Never \
-- bin/kafka-console-producer.sh \
--bootstrap-server my-cluster-kafka-bootstrap:9092 \
--topic quote-requests
```
