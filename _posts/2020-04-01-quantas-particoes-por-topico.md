---
title:  Quantas Partições por Tópico?
author: fabiojose
date:   2020-04-01 22:03:36 -0300
categories: [Artigos, Partition]
tags: [número de partições]
toc: false
---

Umas das grandes dúvidas ao utilizar o Kafka é saber quantas partições são necessárias ao criar novos tópicos. Bem, não existe uma fórmula geral, o que temos é uma aproximação detalhada neste excelente artigo [How to choose the number of topics/partitions in a Kafka cluster?](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster/), escrito por Jun Rao. Outro artigo muito relevante é o [Benchmarking Apache Kafka: 2 Million Writes Per Second](https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines), escrito por Jay Kreps. Nele são revelados resultados importantes sobre a taxa de transferência para _producers_ e _consumers_.

Então, com base no artigo de Jun Rao, temos a fórmula aproximada para determinar o número de partições:

```
  MAX(t/p, t/c)
```

Onde:

- `t`: taxa de transferência desejada
- `p`: taxa de transferência do _producer_
- `c`: taxa de transferência do _consumer_

Como o artigo sugere, o valor para a taxa de transferência do _consumer_ depende de como ele processa os registros e por esse motivo devemos realizar nossas próprias medições ao invés de utilizar o valor-base descrito no artigo de Jay Kreps. Já para o valor referente ao _producer_, podemos tomar como base àquele revelado pelo artigo.

> Uma dica é você realizar todas as medições, assim você também entenderá como é a sua infra kafka.

## Aplicando a fórmula

Primeiro temos de definir qual é a unidade da nossa taxa de transferência, que poderia ser MB/s ou Mensagens/s. Mas as mensagens tem tamanhos variados e utilizá-las nas medições talvez não conduza a resultados realistas, então, `MB/s` é uma boa unidade de medida para aplicação da fórmula.

- Unidade: `MB/s` (megabytes por segundo)

A medição é realizada empregando um _producer_ e um tópico com apenas uma partição. Então, digamos que o resultado para nosso `p`, foi:

```
89 MB/s
```

A medição para taxa de transferência do _consumer_ é similar, ou seja, apenas um tópico com uma única partição. Então, digamos que o valor de `c` é:

```
75 MB/s
```

Imagine que nosso alvo com relação a taxa de transferência seja `450 MB/s`. Aplicando a fórmula de aproximação temos:

```
p = 89
c = 75
t = 450

  MAX(450/89, 450/75)

  MAX(5.1, 6) = 6
```

Portanto, o número de partições é `6`.

> Notadamente, não trata-se de uma fórmula para qualquer caso de uso, porém, agora temos um ponto de partida e não somente números mágicos.

Realize testes e coloque nos comentários quais foram os resultados, suas observações são valiósas.

Até o próximo artigo!