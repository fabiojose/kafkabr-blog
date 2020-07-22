---
title:  Medir Performance de Escrita no Cluster
author: fabiojose
date:   2020-06-05 22:03:36 -0300
categories: [Exemplos de Código, Performance]
tags: [dicas, performance, produtor, snippet]
toc: false
---

Você precisa testar a performance de escrita do seu cluster #kafka?

Faça isto de uma manira simples e assertiva com este comando utilitário que vem na instalação padrão:

```bash
# Medir desempenho na produção de eventos:
#  - registros com 51200 bytes (50KB)
#  - lotes de 400KB cada
kafka-producer-perf-test.sh \
  --topic 'meu_topico' \
  --num-records 1000000 \
  --record-size 51200 \
  --throughput -1 \
  --print-metrics \
  --producer-props \
      'acks=1' \
      'bootstrap.servers=127.0.0.1:9092' \
      'batch.size=409600'
```