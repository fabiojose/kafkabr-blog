---
title: Porque utilizar Apache Kafka®
author: fabiojose
date: 2020-11-22 06:53:37 -0300
categories: [Artigos, Avaliação]
tags: [kafka]
toc: true
image: /assets/img/emily-morter-8xAA0f9yQnE-unsplash.jpg
---

Ao conhecer ou saber da existência do Apache Kafka®, muitas perguntas surgem:

- "Onde usá-lo ..."
- "Como usá-lo ..."
- "Quais problemas ele resolve ..."
- "Como ele funciona ..."
- "Quando usá-lo ..."
- ["Ele é uma fila?"](https://blog.kafkabr.com/posts/kafka-nao-e-fila/)

Todas elas são rapidade respondidas, mas esta ai dá mais trabalho:

- "_Por que_ utilizar o Apache Kafka®?"

Afinal, quais são o motivos que nos levam a usar o Kafka e não outra
ferramenta?

Neste artigo existem elementos para ajudar na responder desta pergunta.

---

## Código Fonte Aberto

- ativamente evoluído por diversas empresas e usuários
- suportado pela Confluent® (projetos _open source_ apadrinhados por grandes
empresas tem um grande alcance, como a Datastax® faz com o Cassandra ou a
Cloudera® com o Hadoop)
- mantido pela fundação Apache®, a maior comunidade de código fonte aberto
do mundo

## Documentação

- __solução de problemas__: pela idade do Apache Kafka® e também pela abrangência
do seu uso, encontramos soluções para muitos problemas. Sejam problemas de
instalação, operação ou desenvolvimento.

- __manuais de uso__: documentação de altíssima qualidade que detalha cada aspecto
do uso do ecossistema _open source_, incluindo: javadoc, documentação oficinal,
documentação Confluent, artigos, exemplos de código e comunidades de usuários.

## Integração

- [várias linguagens](https://cwiki.apache.org/confluence/display/KAFKA/Clients)
com suporte as APIs Producer e Consumer
- [Rest Proxy](https://github.com/confluentinc/kafka-rest)
para as linguagens antigas ou sem suporte nativo
- [Centenas de conectores](https://www.confluent.io/hub/) para __Kafka Connect__
- Suporte avançado nos principais frameworks:
[Spring](https://spring.io/projects/spring-kafka),
[Quarkus](https://quarkus.io/guides/kafka),
[Vertx](https://vertx.io/docs/vertx-kafka-client/java/),
[Micronaut](https://micronaut-projects.github.io/micronaut-kafka/latest/guide/),
[Akka](https://doc.akka.io/docs/alpakka-kafka/current/)
- Replicação multi-datacenter, multi-cloud ou multi-região com 
[Mirror Maker](https://kafka.apache.org/documentation/#basic_ops_mirror_maker)

## Ecossistema

- Producer e Consumer API: simples, idempotentes ou transacionais
- Streaming com Kafka Streams, escrito sobre a API Consumer e Producer
- Integração com origens ou destinos de dados com Kafka Connect
- [Schema Registry](https://github.com/confluentinc/schema-registry)
- [ksqlDB](https://ksqldb.io/)
- milhares de exemplos sobre como utilizar cada parte do ecossistema
- muitas outras [ferramentas e utilitários](https://blog.kafkabr.com/posts/apache-kafka-ferramentas/)

## Casos de Uso
- ideal para processamento de eventos
- arquitetura elaborada para alta performance e alta disponibilidade
- níveis ajustáveis para consistência e confiabilidade
- modelo de persistência durável, que pode ser ajustada em nível de tópico
- consumo concorrente _out-of-the-box_
- garantia de ordem em nível de partição
- consumo por _poll_, o que reduz as responsabilidades do broker
e não satura o consumidores
- nativo para nuvem (_cloud native_)

## Conclusão

Bem, é isso. Estes foram os elementos e eles ajudarão qualquer pessoa a 
responde _Por que utilizar o Apache Kafka®?_.

Até o próximo!

---

<span>Photo by <a href="https://unsplash.com/@emilymorter?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Emily Morter</a> on <a href="https://unsplash.com/s/photos/question?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>