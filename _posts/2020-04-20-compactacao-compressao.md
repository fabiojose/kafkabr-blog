---
title:  Compactação e Compressão no Apache Kafka
author: fabiojose
date:   2020-04-20 22:03:36 -0300
categories: [Artigos, Registro]
tags: [tópico, compressão, compactação]
toc: false
---

Compactar e comprimir tem algo em comum:

- reduzir o espaço ocupado pelos dados

Seja removendo partes desnecessárias através da compactação ou tornado-as menores com algoritmos de compressão.

No Kafka podemos utilizar ambas.

---

## Compactação

A compactação no Kafka pode ser ativada através de uma configuração feita nos tópicos:

- `cleanup.policy=compact`

> Por padrão ela é definida como `delete`. Veja outros detalhes na [documentação oficial](https://kafka.apache.org/documentation/#topicconfigs).

Quando a política de limpeza for igual a `compact`, periodicamente os tópicos são varridos em busca de registros repetidos. E para encontrá-los, o Kafka utiliza a `key` presente em cada um deles.

> Registros sem chave, são imediatamente eliminados.

Acompanhe este exemplo, onde os registros estão dispotos dos mais antigos para os mais novos;

```
Key  Value
c1   Graziela Maciel, MS
c3   Carlos Chagas, MG
c5   Bertha Lutz, SP
c1   Graziela Maciel, RJ
c3   Carlos Chagas, RJ
c5   Bertha Luttz, SP
c7   Suzana Herculano, RJ
c5   Bertha Lutz, RJ
```

Estes são os registros mantidos após o processo de compactação:

```
Key  Value
c1   Graziela Maciel, RJ
c3   Carlos Chagas, RJ
c7   Suzana Herculano, RJ
c5   Bertha Lutz, RJ
```

É assim que funciona a compactação no Kafka, somente o registro mais recente é mantido. Todos os outros mais antigos e com a mesma `key` são eliminados.

Por exemplo, existem três registros com a chave `c5` e o mais recente é `Bertha Lutz, RJ`, que foi mantido pela compactação. Note que haviam 8 registros, restando apenas 4.

## Compressão

A compressão pode ser feita através de uma configuração nos tópicos ou nos produtores.

No __tópico__ a compressão é ativada através da configuração `compression.type`, que por padrão está definida como `producer`. Isso significa que a compressão empregada pelos produtores será mantida e repassada aos consumidores, mesmo que não exista compressão alguma.

Os valores possíveis para `compression.type` são:

- `uncompressed`: mesmo que os produtores enviem dados comprimidos, eles serão persistidos no seu tamanho original, ou seja, sem compressão.
- [`zstd`](https://facebook.github.io/zstd/): algoritmo de compressão criado pelo Facebook.
- [`lz4`](https://github.com/lz4/lz4): a compressão mais recomendada pelos especialistas.
- [`snappy`](https://github.com/google/snappy): algoritmo criado pelo Google.
- `gzip`: ótimos níveis de compressão, usando com mais intensidade a cpu.
- `producer`: mantém a compressão do produtor, se existir.

E no __produtor__ também existe a configuração `compression.type`, que possuí os mesmos valores, com exceção de `uncompressed` e `producer`. Adicionalmente, existe a opção `none` para não aplicar compressão, que também é o valor padrão.

---

Comprimir dados no produtor é interessante porque reduz o uso da banda de rede e de espaço em disco e cpu nos brokers.

E a compressão traz bons resultados quando são produzidos lotes de dados, pois ela não é aplicada em nível de registro. Ou seja, tanto no produtor, quando no Kafka, a compressão não é em cima de cada mensagem ou evento.

---

Já nos consumidores, nenhuma configuração especial é necessária, porque através do protocolo Kafka ele verifica se há compressão e emprega o mesmo algoritmo.