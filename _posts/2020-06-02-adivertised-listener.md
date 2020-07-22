---
title: Advertised Listener
author: fabiojose
date:   2020-06-02 11:03:36 -0300
categories: [Links, Operação]
tags: [cluster, configuração, rmoff, snippet]
toc: false
---

Quando instalamos um broker no cluster #kafka em uma IaaS ou até no #kubernetes, temos que definir corretamente o valor desta configuração. Ela deverá conter host + porta atingíveis pelos clientes kafka.

```properties
# Broker em um provedor de núvem:
#   - Public DNS: para clientes Kafka pela interna
advertised.listeners=PLAINTEXT://ec2-3-89-127-35.compute-1.amazonaws.com:9092

# Vincular a todas as interfaces de rede, dentro de uma EC2 por exemplo
listeners=PLAINTEXT://0.0.0.0:9092
```

Entenda outros detalhes através deste artigo sensacional:

- [My Python/Java/Spring/Go/Whatever Client Won’t Connect to My Apache Kafka Cluster in Docker/AWS/My Brother’s Laptop. Please Help!](https://www.confluent.io/blog/kafka-client-cannot-connect-to-broker-on-aws-on-docker-etc/)