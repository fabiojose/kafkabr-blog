---
title: Apache Kafka® é uma fila?
author: fabiojose
date: 2020-01-10 06:38:37 -0300
categories: [Artigos, Avaliação]
tags: [fila, aprendizado]
toc: true
image: /assets/img/dimitri-houtteman-BjD3KhnTIkg-unsplash.jpg
---

Você já deve ter lido, assistido ou ouvido que o Apache Kafka® é uma fila. Não?

Ou afirmações como estas:

- "Coloca na fila do Kafka!"
- "Le aqui da minha fila no Kafka"
- "Kafka é fila sim!"
- "Ah, mas dá para fazer fila com o Kafka"

Bem, aqui serão listados os motivos e detalhes com o único
objetivo de oferecer a você o entendimento completo sobre como é incorreto
afirmar isso.

Então vamos desmitificar definitivamente sobre porque Kafka não é fila.

## O que é uma fila?

Definição de fila:

- uma estrutura de dados onde objetos são _adicionados_ no final e
_removidos_ do inicio.

Somente a definição acima já deveria remover qualquer idéia sobre
Apache Kafka® ser uma fila.

__Em um tópico Kafka os eventos não são removidos depois de consumidos,
graças a durabilidade.__

### Durabilidade

No Apache Kafka® os eventos depois de produzidos no tópico são mantidos lá por
no mínimo 7 dias, isso em sua
[configuração padrão](https://kafka.apache.org/documentation/#retention.ms).
Depois disso serão eliminados, mesmo que jamais tenham sido consumidos.

Viu? Nenhum comportamento de fila aqui. A durabilidade já é um argumento bem
forte para demover essa idéia.

## Garantia de Ordem

Em uma fila simples todos os dados adicionados são globalmente ordenados,
ou seja, existe a garantia de que a ordem em que foram persistidos será
globalmente mantida.

Um tópico no Apache Kafka® é subdivido em partições e a garantia de ordem é
somente neste nível. Então, tópicos com várias partições não existe ordenação
global entre elas, somente localmente em cada uma.

Os detalhes sobre esta garantia estão no quarto paragráfo da 
[documentação oficial](https://kafka.apache.org/documentation/#intro_concepts_and_terms)
deixando muito claro qual será o comportamento.

Como você deve pensar, tópicos com apenas uma partição tem comportamento similar
ao de fila. Então haverá [a ilusão de que adicionar mais partições irá escalá-la](https://blog.kafkabr.com/posts/erros-comuns-iniciantes/#criar-o-primeiro-t%C3%B3pico-com-apenas-uma-parti%C3%A7%C3%A3o).
Não se engane, é somente uma ilusão mesmo.

## Producer, Lotes e Retentativas

Ao contrário de filas tradicionais, que entregam assim-que-possível,
no Kafka os eventos produzidos são loteados para uso
racional da banda de rede e compressão. Portanto ao serem produzidos, os 
eventos ainda permaneceram na origem por algum tempo.

Este comportamento é definido por configurações no Producer Apache Kafka®:

- [batch.size](https://kafka.apache.org/documentation/#batch.size): tamanho do
lote de eventos
- [linger.ms](https://kafka.apache.org/documentation/#linger.ms): estende
o tempo local para montar lotes de eventos

Outro fato são as retentativas em caso de erro na entrega do lote de eventos
ao broker. Quando isso acontece, automaticamente
haverão reenvios que naturalmente poderão modificar a [ordem entre os vários
lotes](https://blog.kafkabr.com/posts/garantia-de-ordem/) que estão aguardando
reconhecimento de entrega.

Isto é determinado pelas seguintes configurações:

- [retries](https://kafka.apache.org/documentation/#linger.ms): quantos
reenvios tentar em meio a falha de entrega
- [acks](https://kafka.apache.org/documentation/#acks): reconhecimento de
entrega dos eventos ao broker
- [max.in.flight.requests.per.connection](https://kafka.apache.org/documentation/#max.in.flight.requests.per.connection):
quantas entregas aguardarão reconhecimento simultaneamente.
- [delivery.timeout.ms](https://kafka.apache.org/documentation/#delivery.timeout.ms):
tempo máximo até entregar, reenviar ou falhar totalmente

É claro que filas tradicionais também implementam reenvios em caso e erro,
mas jamais haverá mudança da ordem produzida. E isso quer dizer que enquanto
a mensagem A não for entregue, B ficará aguardando, penalizando
drasticamente a taxa de transferência. Mas Kafka não é fila, por isso há
o compromisso com a ordenação estrita.

## Consumer, Partições e Lotes

Apache Kafka® implementa o consumo paralelo e isso escala muito bem. E o [número
máximo de consumidores](https://blog.kafkabr.com/posts/erros-comuns-iniciantes/#ignorar-a-rela%C3%A7%C3%A3o-entre-grupo-de-consumo-e-parti%C3%A7%C3%B5es) em um grupo de consumo está limitado a quantidade
de partições presentes no tópico em questão.

Talvez você se pergunte:

- "Se apenas um consumidor processar todas as partições do tópico, haverá 
comportamento de fila?"
  - Não. Os eventos já foram particionados na produção e no consumo não existe
  possibilidade de ordenação global.

E mesmo após consumidos, os eventos permanecem no tópico. O que é muito
diferente de uma fila, onde eles serão removidos imediatamente após o consumo. 

> Portanto, durante um cenário de erro, jamais produza o evento novamente no
tópico de onde ele foi consumido. Ele será duplicado e não "enfileirado".

E não, [criar mais grupos de consumo](https://blog.kafkabr.com/posts/erros-comuns-iniciantes/#ignorar-a-rela%C3%A7%C3%A3o-entre-grupo-de-consumo-e-parti%C3%A7%C3%B5es)
não vão escalar o processamento. 🤓

Similar ao `batch.size`, o Consumer também possui configurações para definir
lotes de eventos e assim promover uso racional da banda e processamento.
São elas:

- [fetch.min.bytes](https://kafka.apache.org/documentation/#fetch.min.bytes):
quantidade mínima de bytes que devem existir no tópico para serem consumidos
- [max.partition.fetch.bytes](https://kafka.apache.org/documentation/#max.partition.fetch.bytes):
quantidade máxima de bytes a serem lidos de cada partição
- [fetch.max.bytes](https://kafka.apache.org/documentation/#fetch.max.bytes):
quantidade máximo de bytes que devem ser lidos do tópico
- [max.poll.records](https://kafka.apache.org/documentation/#max.poll.records):
quantidade de eventos que devem ser lidos do tópico
- [fetch.max.wait.ms](https://kafka.apache.org/documentation/#fetch.max.wait.ms):
tempo máximo a aguardar até que o mínimo de bytes existam no tópico

Aliado a isso, existe a confirmação de _offset_. Onde o Consumer envia ao 
Kafka a confirmação de que o maior _offset_ em um determinado lote de eventos
já foi processado. Esta confirmação é identificada por:

- grupo de consumo
- nome do tópico
- número da partição

Assim, caso o Consumer falhe outro retoma seu trabalho e segue de onde parou,
por exemplo.

Mas como estamos falando de computação distribuída, isso pode falhar e os
eventos serão consumidos novamente. Por este motivo a idempotência sempre
é discutida e deve ser considerada quando se trabalha com Apache Kafka®.

## Conclusão

Estes foram os fatos sobre como é incorreto afirmar que Apache Kafka® é fila e 
também como é incorreto dizer: 

- "ah, mas tecnicamente é possível se eu criar o tópico só com uma partição"
- "eu posso desligar o loteamento de eventos para produzir e consumir um evento por vez"

Kafka não é fila. Ele implementa uma característica das filas. Que é:

- dado um evento, este será processado por extamente um Consumer
(dentro do grupo de consumo).

Este é um resumo sobre sete itens abordados sobre Apache Kafka®:

1. __Durabilidade__: os eventos são duráveis
2. __Garantia de Ordem__: não existe ordenação global no tópico
3. __Partições__: unidades para paralelizar o consumo e ordenação local
4. __Lotes__: produzir e consumir eventos é sempre em lote
5. __Retentativas__: são executadas de forma transparente
6. __Grupo de Consumo__: é o que faz Kafka ser muito diferente
7. __Idempotência__: sempre tenha isso em mente

Se Kafka fosse fila, ele seria uma outra ferramenta qualquer como várias
que existem por ai. Logo ele não seria tão popular na problemática dos eventos e 
sistemas distribuídos.

---

<span>Photo by <a href="https://unsplash.com/@dimhou?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Dimitri Houtteman</a> on <a href="https://unsplash.com/s/photos/not?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>