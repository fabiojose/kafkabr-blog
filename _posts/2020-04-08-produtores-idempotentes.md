---
title: Produtores Idempotentes
author: fabiojose
date:   2020-04-08 11:03:36 -0300
categories: [Exemplos de Código, Produtor]
tags: [produtor, configuração, snippet]
toc: false
---

Com essas configurações, seu produtor #kafka será idempotente e operando com garantia de entrega exactly-once, ou seja, sem duplicação de eventos no broker. 

```properties
enable.idempotence=true
acks=all
max.in.flight.requests.per.connection=5
```