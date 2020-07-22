---
title: Serialização de Eventos
author: fabiojose
date:   2020-05-22 11:03:36 -0300
categories: [Links, Registro]
tags: [snippet, registro, serializacao, produtor, consumidor]
toc: false
---

Você sabia que no #kafka os eventos são apenas arranjos (array) de bytes? Então o seu tipo, seja ele simples ou complexo, é definido no Producer e interpretado pelo Consumer. Logo, suas configurações deverão ser equivalentes.

```properties
# Serializador no Producer
#    recebendo strings e transformando-as em bytes
value.serializer=org.apache.kafka.common.serialization.StringSerializer

# Deserializador no Consumer
#    recebendo bytes e transformando-os em strings
value.deserializer=org.apache.kafka.common.serialization.StringDeserializer

# Producer => Kafka => Consumer
# String   => Bytes => String
```