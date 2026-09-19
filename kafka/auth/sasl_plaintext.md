---
title: kafka/auth/sasl_plaintext
tags: [kafka]
---

- ### All components involved in the setup must be adapted for authentication:
  - Zookeeper
  - Kafka
  - Kafka Connect
  - Schema Registry (Confluent)
  - akhq
- ### Before adjusting the configs, the configurations of the services must be adjusted:
  - systemctl edit zookeeper.service
    ```service
    [Service]
    Environment=SERVER_JVMFLAGS=\"-Djava.security.auth.login.config=/etc/zookeeper/jaas.conf\"
    ```
  - systemctl edit kafka.service
    ```service
    [Service]
    Environment=\"KAFKA_OPTS=-Djava.security.auth.login.config=/etc/kafka/jaas.conf\"
    ```
  - For Zookeeper and Kafka, the jaas.conf files must be in the right place for the services to start.
- ### The following configuration files must be changed:
  - /opt/kafka/config/server.properties
    ```properties
    # for any
    super.users=User:admin
    authorizer.class.name=kafka.security.authorizer.AclAuthorizer
    sasl.enabled.mechanisms=PLAIN
    sasl.mechanism.inter.broker.protocol=PLAIN
    security.inter.broker.protocol=SASL_PLAINTEXT
    
    # only sasl
    #listeners=SASL_PLAINTEXT://:9092
    #advertised.listeners=SASL_PLAINTEXT://:9092
    #security.protocol=SASL_PLAINTEXT
    
    # multiple listeners
    listeners=PLAINTEXT://:9093,SASL_PLAINTEXT://:9092
    advertised.listeners=PLAINTEXT://:9093,SASL_PLAINTEXT://:9092
    listener.security.protocol.map=SASL_PLAINTEXT:SASL_PLAINTEXT,PLAINTEXT:PLAINTEXT
    ```
  - /opt/kafka/config/jaas.conf
    ```sh
    ln -s /opt/kafka/config/jaas.conf /etc/kafka/jaas.conf
    ```
  - /opt/kafka/config/connect-distributed.properties
    
    ```properties
    
    # add proper listener. listener comes from server.properties
    bootstrap.servers=localhost:9092 
    
    security.protocol=SASL_PLAINTEXT
    sasl.mechanism=PLAIN
    sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
        username="connect-client" \
        password="<redacted>";
    
    producer.security.protocol=SASL_PLAINTEXT
    producer.sasl.mechanism=PLAIN
    producer.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
        username="connect-producer" \
        password="<redacted>";
    
    consumer.security.protocol=SASL_PLAINTEXT
    consumer.sasl.mechanism=PLAIN
    consumer.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
        username="connect-consumer" \
        password="<redacted>";
    ```
  - /etc/schema-registry/schema-registry.properties
    
    ```properties
    listeners=http://0.0.0.0:8085
    kafkastore.bootstrap.servers=SASL_PLAINTEXT://192.168.11.10:9092,\
    SASL_PLAINTEXT://192.168.11.11:9092,\
    SASL_PLAINTEXT://192.168.11.12:9092
    host.name=192.168.11.10
    kafkastore.topic=_schemas
    debug=true
    kafkastore.security.protocol=SASL_PLAINTEXT
    kafkastore.sasl.mechanism=PLAIN
    kafkastore.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
        username="schema" \
        password="<redacted>";
    ```
  - /opt/zookeeper/conf/jaas.conf
    ```sh
    ln -s /opt/zookeeper/conf/jaas.conf /etc/zookeeper/jaas.conf
    ```
  - here jaas config is populated with admin pw from /opt/kafka/config/jaas.conf
    ```conf
    QuorumServer {
        org.apache.zookeeper.server.auth.DigestLoginModule required
        user_admin="<redacted>";
    };
    QuorumLearner {
        org.apache.zookeeper.server.auth.DigestLoginModule required
        username="admin"
        password="<redacted>";
    };
    Server {
        org.apache.zookeeper.server.auth.DigestLoginModule required
        user_admin="<redacted>";
    };
    ```
  - /opt/zookeeper/conf/zoo.cfg
    ```cfg
    authProvider.sasl=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
    quorum.auth.enableSasl=true
    quorum.auth.learnerRequireSasl=true
    quorum.auth.serverRequireSasl=true
    ```
  - /etc/akhq/akhq.yml
    
    ```yml
    connections:
      foo-bar-kafka:
        properties:
          bootstrap.servers: "192.168.11.20:9092,192.168.11.21:9092,192.168.11.22:9092"
          security.protocol: SASL_PLAINTEXT
          sasl.mechanism: PLAIN
          sasl.jaas.config: org.apache.kafka.common.security.plain.PlainLoginModule required username="admin" password="changeme";
    ```
- ### order of operation
  - configuruing kafka services
  - changing the configs
    - starting kafka server and zookeeper
      - ```sh
        systemctl start kafka
        systemctl start zookeeper
        ```
    - once kafka and zookeeper run on all nodes, check if zookeeper works correctly
      - ```sh
        echo stat | nc localhost 2181
        Zookeeper version: 3.7.2-c06c7c8a3e95779d4becb1938b378596e3b420d0, built on 2023-10-06 09:51 UTC
        Clients:
        /0:0:0:0:0:0:0:1:54440[0](queued=0,recved=1,sent=0)
        /127.0.0.1:57202[1](queued=0,recved=733,sent=733)
        /127.0.0.1:57216[1](queued=0,recved=3585,sent=3664)
        Latency min/avg/max: 0/0.5985/42
        Received: 22746
        Sent: 23003
        Connections: 3
        Outstanding: 0
        Zxid: 0x900001756
        Mode: follower
        Node count: 322
        ```
    - Add ACLs to Zookeeper
      - The rules usually come from the customer/developer. They can be imported with the following script. Importing the same rules several times is no problem.
      - you NEED to import ACLs in order for kafka-connect and schema-registry to start correctly when configured with users from `/opt/kafka/config/jaas.conf`
        ```sh
        #!/usr/bin/env bash
        zookeeperhost="localhost"
        zookeeperport="2181"
        /opt/kafka/bin/kafka-acls.sh --authorizer-properties zookeeper.connect=${zookeeperhost}:${zookeeperport} --add --allow-principal User:admin --operation All --topic '*' --group '*'
        /opt/kafka/bin/kafka-acls.sh --authorizer-properties zookeeper.connect=${zookeeperhost}:${zookeeperport} --add --allow-principal User:schema --operation All --topic '*' --group schema-registry
        /opt/kafka/bin/kafka-acls.sh --authorizer-properties zookeeper.connect=${zookeeperhost}:${zookeeperport} --add --allow-principal User:ksc-brokerbridge-transaction-api --operation All --topic '*' --group '*'
        /opt/kafka/bin/kafka-acls.sh --authorizer-properties zookeeper.connect=${zookeeperhost}:${zookeeperport} --add --allow-principal User:ksc-brokerbridge-transaction-service --operation All --topic '*' --group '*'
        .
        .
        .
        ```
  - restarting services
    ```sh
    #!/usr/bin/env bash
    systemctl "${1}" zookeeper.service\
        akhq.service\
        confluent-schema-registry.service\
        kafka-connect.service\
        kafka-ui.service\
        kafka.service
    ```
  - customer/dev tests whether everything works as it should
    - don't hesitate to test things yourself e.g. using nagios
      ```sh
      /usr/bin/docker run --rm -v /usr/lib/nagios/plugins/check_kafka.py:/github/nagios-plugins/check_kafka.py --net=host harisekhon/nagios-plugins check_kafka.py --host localhost --topic nagios --sasl-username admin --sasl-password <redacted>
      /usr/bin/docker run --rm --net=host harisekhon/nagios-plugins check_zookeeper.pl  --host localhost
      /usr/lib/nagios/plugins/check_http -H localhost -p 8085
      /usr/lib/nagios/plugins/check_http -H localhost -p 8083
      ```
  - fin
