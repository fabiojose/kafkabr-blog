---
title:  Consumindo Registros com Spring Boot
author: fabiojose
date:   2020-04-28 22:03:36 -0300
categories: [Artigos, Consumidor]
tags: [spring, java, dicas]
toc: false
---

O consumo de dados do Kafka, dependendo do caso de uso, requer alguns cuidados no commit. Fazê-lo de forma correta é importante para garantir que nada seja perdido.

Com Clientes Kafka, em especial a versão para Java, é muito simples e intuitivo, mas no Spring Kafka é necessária atenção com ajustes especiais.

---

O foco neste artigo é mostrar como consumir dados com Spring Kafka, que é uma abstração sobre os [Clientes Kafka](https://docs.confluent.io/current/clients/java.html#java-client) para Java. Fazendo isso com seguindo setup:

- `enable.auto.commit=false`. Que requer commit manual
- commit síncrono

> Devido a semântica de entrega _at-least-once_, um registro poderá ser consumido uma ou `n` vezes, e este setup busca reduzir isso.

---

Um consumidor típico no Spring Kafka é escrito assim:

```java
@Component
public class SpringKafkaListener {
  @KafkaListener(topics = "topico")
  public void consume(String valor) {
    // Processar valor do registro
  }
}
```

E criado com as seguintes configurações, feitas no `application.properties`:

```properties
spring.kafka.bootstrap-servers=configure-me_kafka-broker:9092
spring.kafka.consumer.client-id=configure-me_client-id
spring.kafka.consumer.group-id=configure-me_group-id
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.StringDeserializer
```

Felizmente o Spring Kafka não redefine valores, portanto a configuração padrão é mantida assim como definida na [documentação oficial](https://kafka.apache.org/documentation/#consumerconfigs), porém, nela o commit é automático e assíncrono. Contudo, no Spring Kafka, todas as configurações necessárias para commit manual e síncrono não estão disponíveis através de propriedades no `application.properties`.

## Resolvendo

Spring Kafka é uma abstração, logo o _poll loop_ e commit são transparentes. E como pode-se ver no exemplo, um consumidor recebe apenas o registro e por padrão não tem acesso ao _Consumer_.

__Primeiro__ é necessário revisar as configurações para desligar o commit automático.

Nova configuração:

```properties
# Nada de novo aqui
spring.kafka.bootstrap-servers=configure-me_kafka-broker:9092
spring.kafka.consumer.client-id=configure-me_client-id
spring.kafka.consumer.group-id=configure-me_group-id
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.StringDeserializer

# Desliga o commit automático no Cliente Kafka
spring.kafka.consumer.enable-auto-commit=false
```

> Spring tem sua própria notação para a maioria das configurações presentes no Kafka Consumer, que são traduzidas em tempo de execução para o nome correto.

Agora que o commit automático foi desligado, são necessários alguns ajustes programáticos feitos ao customizar as fábricas de objetos:

- ackMode para `MANUAL`
- syncCommits como `true`

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.kafka.KafkaProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.config.KafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.ConcurrentMessageListenerContainer;
import org.springframework.kafka.listener.ContainerProperties.AckMode;

@EnableKafka
@Configuration
public class KafkaConfig {

  @Autowired
  KafkaProperties properties;

  @Bean
  public ConsumerFactory<String, String> consumerFactory() {
      return new DefaultKafkaConsumerFactory<>(
              properties.buildConsumerProperties());
  }

  @Bean
  public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String>>
      kafkaListenerContainerFactory() {

      ConcurrentKafkaListenerContainerFactory<String, String> listener = 
            new ConcurrentKafkaListenerContainerFactory<>();
      
      listener.setConsumerFactory(consumerFactory());

      // Não falhar, caso ainda não existam os tópicos para consumo
      listener.getContainerProperties()
          .setMissingTopicsFatal(false);

      // ### AQUI
      // Commit manual do offset
      listener.getContainerProperties().setAckMode(AckMode.MANUAL);

      // ### AQUI
      // Commits síncronos
      listener.getContainerProperties().setSyncCommits(Boolean.TRUE);

      return listener;
    }
}
```

Então o consumidor com Spring Kafka terá esta aparência:

```java
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Component;

@Component
public class SpringKafkaListener {

  @KafkaListener(topics = "topico")
  public void consume(@Payload String valor, Acknowledgment ack) {

    //TODO Processar registro
    // . . . 

    // Commmit manual, que também será síncrono
    ack.acknowledge();

  }
}
```

Note que mesmo assim não existe acesso ao _Consumer_, ao invez disso, o Spring injeta uma instância de `Acknowledgment` que faz o commit quando tem seu método `acknowledge()` executado.

Também neste exemplo o offset é confirmado a cada registro processado. Isso é algo que degrada a taxa de transferência, mas reduz ainda mais as chances de consumos duplicados. Bem, mas cada caso é um caso 😊.

O exemplo completo está disponível no Github:

- [Código fonte no Github](https://github.com/fabiojose/skc-ex)
