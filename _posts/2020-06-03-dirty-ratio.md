---
title: Como Implantar Apache Kafka no Kubernetes - Parte 1
author: fabiojose
date:   2020-06-03 11:03:36 -0300
categories: [Links, Tópico]
tags: [tópico, configuração, snippet]
toc: false
---

A compactação de log no #kafka acontece quando `cleanup.policy=compact`, mas ela está condiciona a outras configurações, como esta. Ela define qual a razão mínima para limpar segmentos das partições.

```properties
## - - - - - - - - Exemplo 1 - - - - - - - - ##
# Compactar quando 50% ou mais do
# segmento contiver duplicatas
min.cleanable.dirty.ratio=0.5

## - - - - - - - - Exemplo 2 - - - - - - - - ##
# Compactar quando 10% ou mais do
# segmento contiver duplicatas
min.cleanable.dirty.ratio=0.1

## - - - - - - - - Exemplo 3 - - - - - - - - ##
# Compactar quando 90% ou mais do
# segmento contiver duplicatas
min.cleanable.dirty.ratio=0.9
```