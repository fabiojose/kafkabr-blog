---
title: Racks na Configuração do Tópico
author: fabiojose
date:   2020-04-22 11:03:36 -0300
categories: [Exemplos de Código, Tópico]
tags: [tópico, configuração, snippet]
toc: false
---

Utilize esta configuração para ativar o rack awareness no #kafka. Ela garante o espalhamento das réplicas de uma partição em locais físicos diferentes, aumentando ainda mais a confiabilidade.

```properties
# -- Exemplos --

# na sua nuvem privada
broker.rack=rack1

# na AWS, região são paulo
broker.rack=sp-east-1

# na AWS, região Virgínia
broker.rack=us-east-1
```