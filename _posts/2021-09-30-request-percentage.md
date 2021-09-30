---
title: Cota para Uso de Tempo em Threads
author: fabiojose
date: 2021-09-25 06:38:37 -0300
categories: [Artigos, Avaliação]
tags: [producer, produtor, consumer, consumidor, cota, quota]
toc: true
image: /assets/img/jason-richard-naK1i6nCuqc-unsplash.jpg
---

- a.k.a. [request_percentage](request_percentage)

Esta é uma cota introduzida pela 
[KIP-124](https://cwiki.apache.org/confluence/display/KAFKA/KIP-124+-+Request+rate+quotas),
e pode ser confusa no primeiro contato.

Ao ler a proposta de melhoria existem exemplos de valores para `request_percentage`,
como o seguinte:

```console
bin/kafka-configs --zookeeper localhost:2181 \
  --alter \
  --add-config 'request_percentage=200' \
  --entity-type users

# 'request_percentage=200' ?????
```

- __200%?__

O entendimento comum por pocentagem é um valor entre 0 e 100, que é
relavito a qualquer notação numérica, seja ela de tempo ou volume de dados.

Exemplos: 

- 20% de 1 segundo é o equivalente a 200 milissegundos
- 10% de 100 bytes é o equivalante a 10 bytes

Bem, mas a cota `request_percentage` utiliza notação absoluta. E neste artigo
existem os detalhes sobre como utilizá-la de maneira efetiva, algo importante
se um cluster Apache Kafka® receberá múltiplos tenants.

## A Cota

A primeira vista parece um erro de digitação encontrar na documentação oficial
um exemplo para `request_percentage` com o valor `200`. Mas ele está lá
propositalmente. E se compreende isso ao ler seção
[Rejected Alternatives](https://cwiki.apache.org/confluence/display/KAFKA/KIP-124+-+Request+rate+quotas#KIP124Requestratequotas-RejectedAlternatives).

Especificamente a opção _Use request time percentage across all threads instead of per-thread percentage for quota bound_.

![](/assets/img/kip.png)

Ela explica porque foi implementada uma solução com valor absoluto. Que, resumindo,
foi escolhida para evitar ajustes automáticos quando `num.io.threads`
ou `num.network.threads` são modificados.

> Absolute quota was chosen to avoid automatic changes to client quota values
> when `num.io.threads` or `num.network.threads` is modified.

Isso quer dizer que `200%` está correto e significa uma cota em duas
threads, ou seja, `100%` da janela de tempo para uso em cada uma.

## Entendedo a Conta da Capacidade Total

A capacidade total é expressada pela seguinte formula:

```
(num.io.threads + num.network.threads) * 100%
```

- (**número de threads para E/S** + **número de threads para rede**) \* **cem porcento**

Então, se `num.io.threads=8` e `num.network.threads=3` haverá a seguinte
capacidade:

```
(8 + 3) * 100
11 * 100
1100
```

Capacidade total de `1100%`. Sim, você leu correto: **mil e cem porcento**

Ou seja, a capacidade total é a soma dos 100% da janela de tempo que cada
thread.

E normalmente as configurações de threads é alinhada com o número total de núcleos
de processamente, assim como a própria KIP-124 descreve. Então se existirem
32 núcleos disponíveis a conta ficará assim:

```
(32 + 32) * 100
64 * 100
6400
```

Isso mesmo!

- `6400%`
- seis mil e quatrocentros porcento

Naturalmente, é 100% da janela de tempo configurada na propriedade
`quota.window.size.seconds`.

## Na Prática

Então se existe as seguinte configuração:

- capacidade total: __6400__
- request_percentage: __200__
- quota.window.size.seconds: __1__

Significa que o `client.id` ou `user` em questão pode utilizar `1s`
em duas threads ao mesmo tempo. E ao execeder isso seu throttling
iniciará.

### Quantidade fixa tenants

Se existem 50 tenants e busca-se distribuir igualitariamente a capacidade
total, então:

``` 
request_percentage=6400 / 50
request_percentage=128
```

Então para cada client.id ou user, `request_percentage` será igual a 128%.
O que dá direito ao uso de duas threads, uma delas 100% da janela de tempo
e 28% (praticamente 1/3) em outra, simultaneamente.

### Quantidade variável de tenants 

Quando a quantidade de tenants e variável e mesmo assim deseja-se
distribuir igualitariamente a capacidade total, será necessário ajustar
o `request_percentage` de todos sempre que um novo for adicionado.

Se existem 100 tenants, a distribuição será a seguinte:

```
request_percentage=6400 / 100
request_percentage=64
```

- para cada client.id ou user, `request_percentage` será igual a 64%.

Ao adicionar mais 15 tenants, aos 100 que existiam, haverá o seguinte:

```
request_percentage=6400 / 115
request_percentage=55.7
```

Então todos os 115 tenants deverão ter seu
`request_percentage` ajustado para __55.7%__.

## Bônus

Este é um script para suportar o ajuste igualitário em um cluster Apache Kafka®
com quantidade variável de tenants.

> [Faça o download](https://gist.github.com/fabiojose/e43c2244d0171926ea590ed6b04ff72b)

Depois de executá-lo, além da capacidade total disponível e o valor de
`request_percentage` por tenant, também será exibido o comando Kafka CLI
para atualização do valor.

Exemplo:

```bash
#                       <kafka>         <tenants>
./request_percentage.sh localhost:19092 120

broker.id...................: 30
num.io.threads..............: 8
num.network.threads.........: 3
quota.window.size.seconds...: 1

capacidade total............: 1100
tenants.....................: 120

request_percentage p/ tenant: 9.2

Comando atualização cota para user default:
  kafka-configs.sh --bootstrap-server localhost:19092 --alter --add-config 'request_percentage=9.2' --entity-type users --entity-default

Comando atualização cota para client.id default:
  kafka-configs.sh --bootstrap-server localhost:19092 --alter --add-config 'request_percentage=9.2' --entity-type clients --entity-default
```

Photo by <a href="https://unsplash.com/@jasonthedesigner?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Jason Richard</a> on <a href="https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
  