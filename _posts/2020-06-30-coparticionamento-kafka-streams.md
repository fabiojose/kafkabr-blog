---
title: Co-Particionamento de Tópicos
author: fabiojose
date:   2020-06-30 11:03:36 -0300
categories: [Links, Kafka Streams]
tags: [ksql, kafka-streams, streaming]
toc: false
---

Em #KafkaStreams e #KSQLDB, uma operação muito comum é a junção, ou join, de tópicos. Para isso ambos devem se co-particionados:

- mesma chave de particionamento
- mesmo número de partições
- mesma estratégia de particionamento

- [Co-partitioning Requirements](https://docs.ksqldb.io/en/latest/developer-guide/joins/partition-data/#co-partitioning-requirements)
