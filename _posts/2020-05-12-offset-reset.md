---
title: Estratégia para Offset
author: fabiojose
date:   2020-05-12 11:03:36 -0300
categories: [Exemplos de Código, Consumidor]
tags: [snippet, configuração, consumidor]
toc: false
---

Quando aplicações se inscrevem nos tópicos do #kafka para consumir eventos é necessário definir uma estratégia que determina a partir de qual offset o broker iniciará o envio ao consumer.

```properties
# A partir do evento mais antigo
#   - o menor offset, 0 por exemplo
auto.offset.reset=earliest

# A partir do evento mais recente
#   - o maior offset, 38999 por exemplo
auto.offset.reset=latest
```