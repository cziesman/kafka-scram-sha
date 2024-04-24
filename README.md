## Kafka SCRAM-SHA-512 Authentication Demo

This project provides a very simple demonstration of a Kafka broker that uses SCRAM-SHA-512 for client authentication and a Java Kafka client that connects to a Kafka broker.

It depends on the *spring-kafka* library to provide the plumbing needed to connect to the Kafka broker and to send and receive messages.

It also uses the *openshift-maven-plugin* to simplify deployment of the demo application to an Openshift project.

### Assumptions

The demo assumes that the _AMQ Streams_ operator is used to manage Kafka, and that the developer can login to Openshift with access to Kafka resources.

The demo application is deployed into its own project. In this case we use *kafka-client* for the project name. Kafka is deployed in the *kafka* project.

The demo application uses a topic named *my-topic*. This topic should be created in Kafka using the _AMQ Streams_ operator.


### Kafka Configuration

In order to allow access to Kafka from outside Openshift, a route needs to be defined. To create the necessary route, update the *listeners* section of the YAML file for the cluster with the following snippet. Since SCRAM-SHA-512 authentication will be used, the listener for PORT 9094 needs to include the authentication details. In this case, the cluster is named *my-cluster*.

      - name: plain
        port: 9092
        tls: false
        type: internal
      - authentication:
          type: scram-sha-512
        name: tls
        port: 9093
        tls: true
        type: internal
      - authentication:
          type: scram-sha-512
        name: external
        port: 9094
        tls: true
        type: route

The name for the listener on port 9094 is arbitrary. Here we use *external* to indicate that the listener is for clients that are external to Openshift.

A Kafka topic called `my-topic` needs to be created with ten partitions. The YAML should appear similar to the following:

    apiVersion: kafka.strimzi.io/v1beta2
    kind: KafkaTopic
    metadata:
      labels:
        strimzi.io/cluster: my-cluster
      name: my-topic
      namespace: kafka
    spec:
      config: {}
      partitions: 10
      replicas: 3

The next step is to create a Kafka User using the _AMQ Streams_ operator. Create a `kafka-user` in the `kafka-cluster`. The authentication type must be `scram-sha-512`. The YAML should appear similar to the following:

    apiVersion: kafka.strimzi.io/v1beta2
    kind: KafkaUser
    metadata:
      name: kafka-user
      labels:
        strimzi.io/cluster: my-cluster
    spec:
      authentication:
        type: scram-sha-512

### User Credentials

Since SCRAM-SHA-512 authentication uses a user name and password, the credentials need to be retrieved from Openshift. When the _AMQ Streams_ operator creates the user `my-user`, it also creates a Secret named `my-user` that contains the password. 

Use the following command to extract the user password.

    oc get secret my-user -n kafka -o jsonpath='{.data.password}' | base64 -d

Use the following command to extract the JAAS configuration.

    oc get secret my-user -n kafka -o jsonpath='{.data.sasl\.jaas\.config}' | base64 -d 

The password and the JAAS configuration will need to be configured in the client application, and will be used to authenticate against the Kafka broker.

### Client Application

The client application is built using Maven, Spring Boot, and the *spring-kafka* library. It consists of a KafkaConsumer and a KafkaProducer.

#### KafkaConsumer

The `KafkaConsumer` uses the `@KafkaLister` annotation, provided by *spring-kafka*, to implement a listener on the `my-topic` topic.

    @KafkaListener(id = "KafkaConsumer", autoStartup = "true", topics = {"${kafka.topic.name}"},
            topicPartitions = @TopicPartition(topic = "${kafka.topic.name}", partitions = "0-9"))
    public void listen(String message) {
        LOG.info(message);
    }

#### KafkaProducer

The `KafkaProducer` uses a `KafkaTemplate`, provided by *spring-kafka*, to send messages to the `my-topic` topic.

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    kafkaTemplate.send(topicName, message);

#### Application Configuration

Spring Boot and *spring-kafka* use pre-defined properties for configuration of Kafka clients, TLS, and authentication. These properties allow Kafka,  TLS, and SCRAM-SHA-512 authentication to be configured without the need to write any code. These properties are located in `application.yaml` and in `application-ocp.yaml` (for Openshift deployment).

    spring:
      application:
        name: kafka-scram-sha
      kafka:
        bootstrap-servers: ${kafka-server}
        enable-auto-commit: true
        jaas:
          enabled: true
        properties:
          sasl:
            mechanism: SCRAM-SHA-512
            jaas:
              config: "org.apache.kafka.common.security.scram.ScramLoginModule required username=\"${kafka.username}\" password=\"${kafka.password}\";"
          security:
            protocol: SASL_SSL
          ssl:
            truststore:
              location: src/main/resources/certs/truststore.p12
              password: ToRmZ8qgDC1H
              type: PKCS12

#### Web User Interface

The application provides a very simple web interface that displays all messages received on the topic *my-topic*. The interface is built using the Thymeleaf library and uses Ajax to update the display dynamically.

When running locally, the web interface can be viewed at:

    http://localhost:9090/web/list

When deployed to Openshift, the web interface can be viewed using a URL similar to the following:

    http://client-kafka-client.apps.cluster-jccqj.dynamic.redhatworkshops.io/web/list

The actual hostname can be retrieved from the Openshift route for the client application.

### Local Deployment

The `application.yaml` file needs to be updated with a few values in order for the application to run successfully.

Set the values for the Kafka bootstrap server.

    kafka-server: my-cluster-kafka-bootstrap-kafka.apps.cluster-jccqj.dynamic.redhatworkshops.io:443

Set the passwords for the new truststore.

        truststore:
          location: src/main/resources/certs/truststore.p12
          password: ToRmZ8qgDC1H
          type: PKCS12

The demo client application is deployed using the following command:

    mvn clean spring-boot:run

This command will compile the code, build a JAR file, and run the JAR file.

Once the application is running, it can be tested using a command similar to the following:

    curl http://localhost:9090/api/send?message=hello

If the message is sent, the response should be `Sent [hello]`.

### Openshift Deployment

The openshift-maven-plugin uses the file `src/main/jkube/deploymentconfig.yaml` to extract the data from the *kafka-client-secret* and to mount the truststore file at a filesystem location where they are accessible by the demo client application.

The demo client application is deployed using the following command:

    mvn clean oc:build oc:deploy -Popenshift

This command will run an S2I build on Openshift to compile the code, build a JAR file, create an image, push the image to Openshift, and deploy the image.

Once the application is running, it can be tested using a command similar to the following:

    curl http://scram-sha-demo-kafka-client.apps.cluster-jccqj.dynamic.redhatworkshops.io/api/send?message=hello

The actual hostname can be retrieved from the Openshift route for the client application.

If the message is sent, the response should be `Sent [hello]`.

