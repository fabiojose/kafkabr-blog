---
title: CloudEvents com Apache Kafka®
author: fabiojose
date:   2020-09-22 05:51:37 -0300
categories: [Artigos, Eventing]
tags: [cloudevents, padrões, eventos, eventing]
toc: false
image: /assets/img/sam-schooler-E9aetBe2w40-unsplash.jpg
---

Assim como o _open source_, os padrões abertos são criados, evoluídos e lançados
por um comunidade de pessoas ou empresas que buscam não depender de licenças pagas
e/ou padrões engessados.

Dentre tantos padrões maravilhosos como: 

- [OpenTracing](https://opentracing.io/)
- [OpenAPI](https://www.openapis.org/)
- [OpenMetrics](https://openmetrics.io/)
- [OpenTelemetry](https://opentelemetry.io/)
- [OpenContainers](https://opencontainers.org/)
- [JSON](https://www.json.org/json-en.html)

Eis que surge o [CloudEvents](https://cloudevents.io/), para suprir a falta de
um padrão para a definicação de eventos na nuvem.

---

Se você trabalha com eventos, arquitetura orientada à eventos ou é uma empresa
orientada à dados, você precisa conhecer este padrão. Ou melhor, você deve 
adotar CloudEvents!

Durante quase dois anos eu fui voluntário no grupo de trabalho responsável pela
definição deste padrão e pela construção das SDKs. Atuei especialmente na
especificação [Avro](https://github.com/cloudevents/spec/blob/v1.0/avro-format.md)
e nas SDKs [JavaScript](https://github.com/cloudevents/sdk-javascript) e
[Java](https://github.com/cloudevents/sdk-java).

## O Padrão CloudEvents

Este padrão define o atributos comuns a qualquer evento, os
[Atributos de Contexto](https://github.com/cloudevents/spec/blob/v1.0/spec.md#context-attributes). Eles podem ser requeridos ou opcionais.
E na versão atual, a `1.0`, são exatamente quatro atributos requeridos:

- [id](https://github.com/cloudevents/spec/blob/v1.0/spec.md#id): idenficiador
único do evento
- [source](https://github.com/cloudevents/spec/blob/v1.0/spec.md#source-1):
fonte do evento, ou seja, qual aplicação o produziu
- [specversion](https://github.com/cloudevents/spec/blob/v1.0/spec.md#specversion):
versão da especificação utilizada, `1.0` por exemplo.
- [type](https://github.com/cloudevents/spec/blob/v1.0/spec.md#type): tipo do evento

Existem muitos outros atributos opcionais, mas o `data` em especial é aquele que 
carrega o _payload_ do evento. Nele encontra-se a carga útil do fato
gerado nas aplicações.

E foram definidos dois modos para codificar um evento:

- Estruturado
- Binário

No modo __Estruturado__ o evento é representado inteiramente no corpo do formato de dados
utilizado, ou seja, seus atributos e carga útil seguem na estrutura.

E no __Binário__, os atributos são cabeçalhos e a carga útil segue sozinha no corpo do formato de dados.

## Exemplos de Eventos

Tome como exemplo o fato gerado por uma transação financeira que alterou o saldo
na conta de um cliente. Ela, depois de confirmada, torna-se um fato e irá compor
o balanço financeiro, ou melhor, o extrato.

__Definição do Evento__

- tipo: Débito Executado ou Crédito Executado
- identificador da transação
- valor da transação
- sistema origem
- código da transação 
- número da conta
- data-hora da transação
- descrição da transação

Na definição do evento estão todos os atributos necessários para que o fato
seja interpretado por qualquer interessado. E alguns deles fazem parte
da carga útil do evento, outros não.

A transação também foi especializada em débito ou crédito, assim os interessados
não deverão aplicar lógicas adicionais sobre o atributo "valor da transação"
para determinar isso.

Bem, veja como este evento fica quando modelado com CloudEvents.

- `type`: Débito Realizado ou Crédito Realizado
- `id`: identificador da transação
- `source`: sistema origem
- `subject`: código a transação
- `time`: data-hora da transação

O restante das características fazem parte da carga útil:

- `data`
  - valor da transação
  - número da conta
  - descrição da transação

Sempre haverão fortes tendências a colocar todas as características
na carga útil do evento, mas sempre tenha em mente que o CloudEvents
foi criado com o objetivo de resolver parte disso. Algo que acontece
quando se inicia o uso de nova especificação, mas que deve ser resolvida
logo no inicio.

No modo __estruturado__ e [formato JSON](https://github.com/cloudevents/spec/blob/v1.0/json-format.md), o evento Débito Executado fica assim:

```json
{
  "type":"ml.kafka.tef.debito.v1",
  "id":"a4c15cfb-dc65-4215-9281-aa5fd50201ab",
  "source":"https://kafka.ml/tef",
  "subject":"890344",
  "time":"2020-09-18T05:31:00Z",
  "specversion":"1.0",
  "datacontenttype": "application/json",
  "data":{
    "valor": -45.89,
    "conta": "234559",
    "descricao": "Pagamento de Impostos"
  }
}
```

Existem dois atributos adicionais:

- `specversion`: define a versão da especificação utilizada
- `datacontenttype`: tipo do conteúdo em `data`, porque ele poderia
ser de qualquer tipo e não exatamente JSON, como o restante do evento.

Então o evento poderá ser emitido através de diversos vínculos de transporte, como:

- [HTTP](https://github.com/cloudevents/spec/blob/v1.0/http-protocol-binding.md)
- [Apache Kafka®](https://github.com/cloudevents/spec/blob/v1.0/kafka-protocol-binding.md)
- [MQTT](https://github.com/cloudevents/spec/blob/v1.0/mqtt-protocol-binding.md)
- [AMQP](https://github.com/cloudevents/spec/blob/v1.0/amqp-protocol-binding.md)
- [NATS](https://github.com/cloudevents/spec/blob/v1.0/nats-protocol-binding.md)

## CloudEvents com Apache Kafka®

> Apache Kafka® foi criado para _streaming_ de eventos, nada mais natural que 
empregar [CloudEvents](https://github.com/cloudevents/spec/blob/v1.0/kafka-protocol-binding.md).

Como o formato de dados [Avro](https://github.com/cloudevents/spec/blob/v1.0/avro-format.md) é um dos mais utilizados, ele será objeto dos exemplos. E o vínculo de transporte, naturalmente, será a especificação CloudEvents para 
[Apache Kafka®](https://github.com/cloudevents/spec/blob/v1.0/kafka-protocol-binding.md).

Também será utilizado o modo [binário](https://github.com/cloudevents/spec/blob/v1.0/kafka-protocol-binding.md#32-binary-content-mode), o que simplifica muito a operação.

E mesmo no modo [estruturado](https://github.com/cloudevents/spec/blob/v1.0/kafka-protocol-binding.md#33-structured-content-mode) os atributos do evento e o _payload_ `data` podem, por definição, serem de tipos diferentes. Ou seja, pode-se utilizar Avro para os atributos CloudEvents e JSON como formato para `data`.

O esquema Avro para a transação de débido ou crédito:

```json
{
    "name":"DebitoExecutado",
    "namespace":"com.kafkabr.e5o",
    "type":"record",
    "version":"1",
    "fields":[
        {
            "name":"valor",
            "type":"double"
        },
        {
            "name":"conta",
            "type":"string"
        },
        {
            "name":"descricao",
            "type":"string"
        }
    ]
}
```

Através de uma aplicação, feita em Java por exemplo, pode-se produzir este
evento como um [ProducerRecord](https://kafka.apache.org/26/javadoc/index.html?org/apache/kafka/clients/producer/KafkaProducer.html).

Primeiro definindo os __Dados do Eventos__ como valor do registro:

```java
DebitoExecutado debito = DebitoExecutado.newBuilder()
    .setValor(-45.89)
    .setConta("234559")
    .setDescricao("Pagamento de Impostos")
    .build();
```

E por segundo os __Atributos CloudEvents__ como cabeçalhos do registro:

> Na [SDK oficial](https://github.com/cloudevents/sdk-java),
o formato Avro ainda não foi implemetado.

```java
  ProducerRecord<String, DebitoExecutado> registro =
    new ProducerRecord<>("transacoes", debito.getConta(), debito);

  registro.headers().add("ce_type",
    "ml.kafka.tef.debito.v1".getBytes());

  registro.headers().add("ce_id",
    "a4c15cfb-dc65-4215-9281-aa5fd50201ab".getBytes());

  registro.headers().add("ce_source",
    "https://kafka.ml/tef".getBytes());

  registro.headers().add("ce_subject",
    "890344".getBytes());

  registro.headers().add("ce_time",
    "2020-09-18T05:31:00Z".getBytes());

  registro.headers().add("ce_specversion",
    "1.0".getBytes());

  registro.headers().add("content-type",
    "application/avro".getBytes());
```

E por fim, só resta produzir o evento no Apache Kafka®:

```java
  try(KafkaProducer<String, DebitoExecutado> producer =
        new KafkaProducer<>(configs)){

      // ####
      // Produzir evento
      producer.send(registro);
  }
```

Deste momento em diante seu ecossistema de serviços baseado em eventos já está
em conformidade com um padrão aberto. E está preparado para as evoluções ou novas
ferramentas que surgirão, ou seja, é uma grande contribuição para o seu presente
e futuro.

## Código Fonte

Como de costume, o código-fonte completo encontra-se no GitHub do Kafka BR:

- [https://github.com/kafkabr/cloudevents-avro](https://github.com/kafkabr/cloudevents-avro)

_Obrigado e até o próximo artigo!_

---
<span>Photo by <a href="https://unsplash.com/@sam?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Sam Schooler</a> on <a href="https://unsplash.com/s/photos/cloud?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>