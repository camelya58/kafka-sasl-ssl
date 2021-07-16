# kafka-sasl-ssl
Instruction for using Apache Kafka with SASL authentication.

## Generate SSL key and certificate for each Kafka broker
The first step of deploying SSL is to generate the key and the certificate for each machine in the cluster. 
```
keytool -keystore kafka.server.keystore.jks -alias localhost -validity 365 -genkey -keyalg RSA
```
You need to specify two parameters in the above command:

- alias: you can use domain name.
- keystore: the keystore file that stores the certificate. The keystore file contains the private key of the certificate; therefore, it needs to be kept safely.
- validity: the valid time of the certificate in days.

Ensure that common name (CN) matches exactly with the fully qualified domain name (FQDN) of the server. The client compares the CN with the DNS domain name to ensure that it is indeed connecting to the desired server, not a malicious one.

## Creating your own CA
After the first step, each machine in the cluster has a public-private key pair, and a certificate to identify the machine. The certificate, however, is unsigned.

A certificate authority (CA) is responsible for signing certificates. 
```
openssl req -new -x509 -keyout ca-key -out ca-cert -days 365
```
The generated CA is simply a public-private key pair and certificate, and it is intended to sign other certificates.

The next step is to add the generated CA to the clients’ truststore so that the clients can trust this CA:
```
keytool -keystore kafka.server.truststore.jks -alias CARoot -import -file ca-cert

keytool -keystore kafka.client.truststore.jks -alias CARoot -import -file ca-cert
```

## Signing the certificate
The next step is to sign all certificates in the keystore with the CA we generated. First, you need to export the certificate from the keystore:
```
keytool -keystore kafka.server.keystore.jks -alias localhost -certreq -file cert-file
```
Then sign it with the CA:
```
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file -out cert-signed -days 365 -CAcreateserial -passin pass:{ca-password}
```
Finally, you need to import both the certificate of the CA and the signed certificate into the keystore:
```
keytool -keystore kafka.server.keystore.jks -alias CARoot -import -file ca-cert

keytool -keystore kafka.server.keystore.jks -alias localhost -import -file cert-signed
```
The definitions of the parameters are the following:

- keystore: the location of the keystore
- ca-cert: the certificate of the CA
- ca-key: the private key of the CA
- ca-password: the passphrase of the CA
- cert-file: the exported, unsigned certificate of the server
- cert-signed: the signed certificate of the server

## SASL Authentication
Sample ${kafka-home}/config/kafka_server_jass.conf file
```java
KafkaServer {
   org.apache.kafka.common.security.scram.ScramLoginModule required
   username="admin"
   password="admin123";
};
```
User credentials for the SCRAM mechanism are stored in ZooKeeper. The kafka-configs.sh tool can be used to manage them
```
 ./kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-512=[password='admin123']' --entity-type users --entity-name admin
Warning: --zookeeper is deprecated and will be removed in a future version of Kafka.
Use --bootstrap-server instead to specify a broker to connect to.
Completed updating config for entity: user-principal 'admin'.
```
Add to ${kafka-home}/config/server.properties file
```
broker.id=0
listeners=SASL_SSL://localhost:9092
advertised.listeners=SASL_SSL://localhost:9092

security.protocol=SASL_SSL
security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512
ssl.endpoint.identification.algorithm=
ssl.client.auth=required
ssl.enabled.protocols=TLSv1.2,TLSv1.1
ssl.secure.random.implementation=SHA1PRNG

# Broker security settings
ssl.truststore.location=C:\\kafka_2.13-2.8.0\\config\\truststore\\kafka.server.truststore.jks
ssl.truststore.password=password
ssl.keystore.location=C:\\kafka_2.13-2.8.0\\config\\keystore\\kafka.server.keystore.jks
ssl.keystore.password=password
ssl.key.password=passord

# ACLs
authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer
super.users=User:admin

#zookeeper SASL
zookeeper.set.acl=false

# With SASL & SSL encryption
scram-sha-512.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="admin" \
  password="admin123";
```

## Start Zookeper
```
.\zookeeper-server-start.sh ..\config\zookeeper.properties
or
.\windows\zookeeper-server-start.bat ..\config\zookeeper.properties
```

## Start Kafka with JAAS
```
export KAFKA_OPTS=-Djava.security.auth.login.config=C:\kafka_2.13-2.8.0\config\kafka_server_jaas.conf
.\kafka-server-start.sh ..\config\server.properties
```
For Windows:
Add to kafka-server-start.bat:
```
IF ["%KAFKA_OPTS%"] EQU [""] (
  set KAFKA_OPTS=-Djava.security.auth.login.config=file:%~dp0../../config/kafka_server_jaas.conf
)
```
and start:
```
.\windows\kafka-server-start.bat ..\config\server.properties
```

## Add to application.yml.
```yml
spring:
  kafka:
    hostname: localhost
    port: 9092
    bootstrapAddress: ${spring.kafka.hostname}:${spring.kafka.port}
    consumer:
      group-id: group
    properties:
      security:
        protocol: SASL_SSL
      sasl:
        mechanism: SCRAM-SHA-512
        jaas:
          config: org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin123";
    ssl:
      trust-store-location:\kafka_2.13-2.8.0\config\truststore\kafka.client.truststore.jks
      trust-store-password:
    jaas:
      enabled: true
```

## Create Kafka producer configs
```java
@Configuration
public class KafkaProducerConfig {

   @Value(value = "${spring.kafka.bootstrapAddress}")
   private String kafkaServer;

   @Value(value = "${spring.kafka.properties.sasl.mechanism}")
   private String mechanism;

   @Value(value = "${spring.kafka.properties.security.protocol}")
   private String protocol;

   @Value(value = "${spring.kafka.ssl.trust-store-location}")
   private String location;

   @Value(value = "${spring.kafka.ssl.trust-store-password}")
   private String trustStorePassword;

   @Value(value = "${spring.kafka.properties.sasl.jaas.config}")
   private String jaasConfig;

    @Bean
    public Map<String, Object> producerConfigs() {
        Map<String, Object> props = new HashMap<>();
        //list of host:port used for establishing the initial connections to the Kafka cluster
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaServer);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(CommonClientConfigs.SECURITY_PROTOCOL_CONFIG, protocol);
        props.put(SaslConfigs.SASL_MECHANISM, mechanism);
        props.put(SaslConfigs.SASL_JAAS_CONFIG, jaasConfig);
        props.put(SslConfigs.SSL_TRUSTSTORE_LOCATION_CONFIG, location);
        props.put(SslConfigs.SSL_TRUSTSTORE_PASSWORD_CONFIG, trustStorePassword);
        return props;
    }

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        return new DefaultKafkaProducerFactory<>(producerConfigs());
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

## Create Kafka consumer configs
```java
@Configuration
@EnableKafka
public class KafkaConsumerConfig {
    
   @Value(value = "${spring.kafka.bootstrapAddress}")
   private String kafkaServer;
   
   @Value("${spring.kafka.consumer.group-id}")
   private String kafkaGroupId;

   @Value(value = "${spring.kafka.properties.sasl.mechanism}")
   private String mechanism;

   @Value(value = "${spring.kafka.properties.security.protocol}")
   private String protocol;

   @Value(value = "${spring.kafka.ssl.trust-store-location}")
   private String location;

   @Value(value = "${spring.kafka.ssl.trust-store-password}")
   private String trustStorePassword;

   @Value(value = "${spring.kafka.properties.sasl.jaas.config}")
   private String jaasConfig;


    @Bean
    public Map<String, Object> consumerConfigs() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaServer);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, kafkaGroupId);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(CommonClientConfigs.SECURITY_PROTOCOL_CONFIG, protocol);
        props.put(SaslConfigs.SASL_MECHANISM, mechanism);
        props.put(SaslConfigs.SASL_JAAS_CONFIG, jaasConfig);
        props.put(SslConfigs.SSL_TRUSTSTORE_LOCATION_CONFIG, location);
        props.put(SslConfigs.SSL_TRUSTSTORE_PASSWORD_CONFIG, trustStorePassword);
        return props;
    }

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        return new DefaultKafkaConsumerFactory<>(consumerConfigs());
    }

    @Bean
    public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String>> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
}
```
For more information see the [article](https://dzone.com/articles/kafka-security-with-sasl-and-acl).
