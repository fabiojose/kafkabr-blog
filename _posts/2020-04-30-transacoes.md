---
title: Transações nos Produtores
author: fabiojose
date:   2020-04-30 11:03:36 -0300
categories: [Exemplos de Código, Produtor]
tags: [produtor, configuração, snippet, consumidor, transação]
toc: false
---

Sabia que o #kafka tem transações? Com ela temos a garantia de que examente todos os registros produzidos serão efetivados. Ou exatamente nenhum em caso de erro.

```properties
# Configurações no produtor
enable.idempotence=true
acks=all
transactional.id=meu-id-global-tx

# Configurações no consumidor
isolation.level=read_committed
enable.auto.commit=false
```