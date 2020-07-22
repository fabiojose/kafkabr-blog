---
title: Compressão ao Produzir Eventos
author: fabiojose
date:   2020-04-13 11:03:36 -0300
categories: [Exemplos de Código, Produtor]
tags: [produtor, configuração, snippet]
toc: false
---

Esta configuração existe no Produtor e no Tópico #kafka. Mantendo o valor padrão no Tópico, o custo computacional da compressão ficará a cargo do produtor e dos consumidores.

```properties
# compressão no Produtor
compression.type=lz4

# compressão padrão no Tópico, que mantém àquela criada pelo produtor
compression.type=producer
```