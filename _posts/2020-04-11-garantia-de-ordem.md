---
title:  Garantia de Ordem
author: fabiojose
date:   2020-04-11 22:03:36 -0300
categories: [Artigos, Producer]
tags: [event sourcing, ordenação, retentativa]
toc: false
---

Existem casos de uso onde a manutenção da ordem em que os dados foram produzidos é um requisito. Há outros em que isso não importa.

---

[Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) é um exemplo onde a manutenção da ordem importa, caso contrário o estado final seria inconsistente.

---

Kafka, através das configurações corretas, pode nos ajudar em qualquer um deles.

## Como é a ordenação no Kafka

A ordem é garantida em nível de partição, ou seja, tópicos com várias partições não possuem ordenação global dos dados.

Bem, dado um lote de registros produzidos pelo produtor `p1` que tem como destino a partição `4` no tópico `t1`, haverá garantia de que a ordem produzida será mantida pelo Kafka e repassada aos consumidores.

Desse modo se houver um produtor `p2` também produzindo lotes de registros no mesmo destino de `p1`, nenhuma verificação de ordem será realizada entre os lotes destes produtores, prevalecendo àquele que primeiro chegar.

Mas existem situações e configurações que podem interferir e mudar a ordenação dos registros criados por um determinado produtor `p`.

## O que pode mudar a ordenação

Há uma dezena de configurações que podem ser ajustadas para determinar como será o comportamento do nosso produtor. E muitas já possuem valores padrão que atendem alguns casos de uso, como:

- retries: 2<sup>16</sup>
  - número de retentativas
- max.in.flight.requests.per.connection: 5 
  - requisições que ainda não receberam ack, ou seja, estão em voo.
- acks: 1
  - reconhecimento de recebimento e não

O valor padrão de retentativas, definido por `retries`, é bem grande e elas são realizadas para todos os problemas relativamente recuperáveis, como uma falha na rede.

E o número de requisições em voo é igual a `5`. E isso poderá causar a troca de ordem.

Digamos que se produz o primeiro lote de registros, `l1`, que fica em voo aguardando reconhecimento. Em seguida o mesmo produtor envia o segundo lote, `l2`. Pela ordenação, `l1` contém registros que estão antes daqueles em `l2`.

Essa é a ordem criada pelo produtor:
```
l1->1,2,3,4
l2->5,6,7
```

Espera-se que seja mantida:
```
1,2,3,4,5,6,7
```

Porém, se `l1` for para retentativa,  `l2` for efetivado e `l1` efetivado em seguida, a ordenação muda, não prevalecendo àquela criada no produtor.

Realidade como descrito acima:
```
5,6,7,1,2,3,4
```

## Configuração correta

Felizmente é possível adequar as configurações do produtor e eliminar as chances da perda de ordem:

- `max.in.flight.requests.per.connection` deverá ser configurado para que apenas uma requisição fique em voo, ou seja, `1`.

Isso certamente irá diminuir a taxa de transferência, não de maneira drástica. Mas agora a ordenação está garantida, mesmo em situações de retentativas.