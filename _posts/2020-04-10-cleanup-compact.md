---
title: Compactação de Log
author: fabiojose
date:   2020-04-1O 11:03:36 -0300
categories: [Exemplos de Código, Tópico]
tags: [tópico, configuração, snippet]
toc: false
---

Use esta configuração nos tópicos do #kafka, então será obrigatório produzir eventos com a chave de particionamento. E, periodicamente, as duplicatas serão removidas, mantendo somente o registro mais recente para cada chave.

```properties
cleanup.policy=compact
```