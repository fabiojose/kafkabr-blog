---
title: Limpeza dos Tópicos
author: fabiojose
date:   2020-05-08 11:03:36 -0300
categories: [Exemplos de Código, Tópico]
tags: [tópico, configuração, snippet]
toc: false
---

No #kafka não existe funcionalidade para apagar os eventos. O que temos é a polícia de limpeza configurada em nível tópico, que está relacionado com a retenção: `retention.ms` e `retention.bytes`.

```properties
# Limpeza por eliminação
cleanup.policy=delete

# Limpeza por compactação
cleanup.policy=compact

# Ambas
cleanup.policy=delete,compact
```