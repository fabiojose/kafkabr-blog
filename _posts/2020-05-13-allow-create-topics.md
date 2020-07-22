---
title: Segurança no Produtor e no Consumidor
author: fabiojose
date:   2020-05-13 11:03:36 -0300
categories: [Exemplos de Código, Consumidor]
tags: [snippet, configuração, consumidor, tópico]
toc: false
---

Esta configuração, feita nos consumidores #kafka, define a estratégia quando o tópico subscrito não existe. Isso pode confundir, porque o tópico só é criado automaticamente se o broker estiver com esta abordagem configurada.

```properties
# -------------- Consumer

# Valor padrão, que terá efeito se o broker permitir
allow.auto.create.topics=true

# Tópico já deverá existir
allow.auto.create.topics=false

# -------------- Broker

## -- Habilita criação automática -- ##
auto.create.topics.enable=true
```