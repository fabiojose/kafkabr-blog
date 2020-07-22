---
title: Retenção de Eventos nos Tópicos
author: fabiojose
date:   2020-05-06 11:03:36 -0300
categories: [Exemplos de Código, Tópico]
tags: [tópico, configuração, snippet]
toc: false
---

No #kafka a retenção dos eventos é definida em nível de tópico, tipicamente configurada por tempo. Mas também é possível defini-la por tamanho ou até ambas.

```properties
# Configuração padrão por tempo: 7 dias
retention.ms=604800000

# -------- Exemplos -------- #

# Reter eventos para sempre
retention.ms=-1

# Retenção por tamanho da partição, ex: 10GB
retention.bytes=10737418240
```