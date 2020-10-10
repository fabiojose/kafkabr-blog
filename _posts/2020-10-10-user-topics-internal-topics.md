---
title: Tópicos Internos vs. Tópicos de Usuário
author: fabiojose
date:   2020-10-10 05:26:37 -0300
categories: [Artigos, Comparativo]
tags: [tópicos, kafka-streams, changelog, rocksdb, segurança]
toc: false
image: /assets/img/jonas-svidras-e28-krnIVmo-unsplash.jpg
---

Os __tópicos internos__, ou _internal topics_ em inglês, são tópicos especiais
criados automaticamente no Apache Kafka® para persistência de _offsets_,
_schemas_, para manunteção do estado das operações sobre streams e até
para o gerencimento de [transações](https://blog.kafkabr.com/posts/transacoes/).

Exemplos:

- `__consumer_offsets`
- `__transaction_state`
- `__schemas`: somente com [Schema Registry](https://docs.confluent.io/current/schema-registry/index.html)
- `{consumer-group}--KSTREAM-JOINOTHER-0000000005-store-changelog`

E os __tópicos de usuários__, ou _user topics_ em inglês, são aqueles criados
para persitência dos eventos das aplicações. Exemplo:

```bash
kafka-topics.sh --create \  
  --bootstrap-server 'localhost:9092' \
  --replication-factor 1 \
  --partitions 7 \
  --topic 'meu-topico-teste' \
  --config 'retention.ms=-1'
```

Ambos podem ser consumidos por qualquer aplicação interessada ou serem
descritos com o `kafka-topics`:

```bash
kafka-topics.sh --describe \
    --topic '__consumer_offsets' \
    --bootstrap-server 'localhost:9092'

# Exemplo da saída
Topic: __consumer_offsets	PartitionCount: 50	ReplicationFactor: 1	Configs: compression.type=producer,cleanup.policy=compact,segment.bytes=104857600
	Topic: __consumer_offsets	Partition: 0	Leader: 1	Replicas: 1	Isr: 1
	Topic: __consumer_offsets	Partition: 1	Leader: 1	Replicas: 1	Isr: 1
	Topic: __consumer_offsets	Partition: 2	Leader: 1	Replicas: 1	Isr: 1
	Topic: __consumer_offsets	Partition: 3	Leader: 1	Replicas: 1	Isr: 1
	Topic: __consumer_offsets	Partition: 4	Leader: 1	Replicas: 1	Isr: 1
	Topic: __consumer_offsets	Partition: 5	Leader: 1	Replicas: 1	Isr: 1
	Topic: __consumer_offsets	Partition: 6	Leader: 1	Replicas: 1	Isr: 1
# . . .
```

Por fim, Kafka Streams faz uso massivo de tópicos internos e isto significa que se
a segurança estiver habilitada no cluster Apache Kafka®, será necessário
conceder permissões de Admin.

> Veja como conceder essas permissões [nesta documentação](https://docs.confluent.io/current/streams/developer-guide/security.html#streams-developer-guide-security) 

_Até os próximos comparativos e avaliações!_

---

<span>Photo by <a href="https://unsplash.com/@jonassvidras?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Jonas Svidras</a> on <a href="https://unsplash.com/s/photos/internal?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>