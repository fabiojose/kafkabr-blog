---
title: Requisições em Voô
author: fabiojose
date:   2020-04-16 11:03:36 -0300
categories: [Exemplos de Código, Produtor]
tags: [produtor, configuração, snippet]
toc: false
---

Aplique esta configuração no seu producer #kafka se o caso de uso requer a manutenção da ordem em que os eventos são produzidos. Naturalmente, se houverem diversos produtores na mesma partição, não haverá garantia de ordem entre eles.

```properties
max.in.flight.requests.per.connection=1
```