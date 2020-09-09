---
title: Cotas de Cliente no Apache Kafka® 2.6.0
author: fabiojose
date:   2020-09-09 05:50:36 -0300
categories: [Artigos, Avaliação]
tags: [produtor, consumidor, admin-api, cotas, kip]
toc: false
image: /assets/img/denys-nevozhai-7nrsVjvALnA-unsplash.jpg
---

O Apache Kafka® sempre teve uma forma para determinar cotas
para produtores e consumidores, os conhecidos _Clients Kafka_. Estas
configurações eram feitas diretamente no Zookeeper e a partir da versão
[2.6.0](https://www.confluent.io/blog/apache-kafka-2-6-updates/#kip-546)
isso foi melhorado.

---

A [KIP-546](https://cwiki.apache.org/confluence/display/KAFKA/KIP-546%3A+Add+Client+Quota+APIs+to+the+Admin+Client) define como deve ser a nova implementação da Admin API e um novo comando 
para o Kafka CLI, o `kafka-client-quotas`.

---

Vejamos como definir cotas para o usuário `conta_corrente`.

# Antes

O comando é um pouco confuso e altamente propenso a erros, além da configuração
ser diretamente aplicada no Zookeeper. E como sabemos, na versão 3.0 do Apache Kafka®
ele será [totalmente removido](https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A+Replace+ZooKeeper+with+a+Self-Managed+Metadata+Quorum) e substituído por uma 
[implementação baseada no protocolo Raft](https://cwiki.apache.org/confluence/display/KAFKA/KIP-595%3A+A+Raft+Protocol+for+the+Metadata+Quorum). 

```bash
kafka-configs.sh --zookeeper localhost:2181 \
                 --alter \
                 --add-config 'consumer_byte_rate=2048' \
                 --entity-type users \
                 --entity-name conta_corrente
```

# Agora

Sem depender diretamente do Zookeeper, agora executamos todas as interações através 
do broker. Ou seja, mais um passo dado em direção a sua remoção.

Além de ser mais intutivo.

```bash
kafka-client-quotas.sh --bootstrap-server localhost:9092 \
					   --alter 
                       --names='user=conta_corrente' \
                       --add='consumer_byte_rate=2048'
```

---

<span>Photo by <a href="https://unsplash.com/@dnevozhai?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Denys Nevozhai</a> on <a href="https://unsplash.com/s/photos/network-traffic?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>