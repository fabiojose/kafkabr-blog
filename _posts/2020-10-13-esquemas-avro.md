---
title: Versionamento de Esquemas Avro com Schema Registry
author: fabiojose
date: 2020-10-13 05:56:37 -0300
categories: [Artigos, Avro]
tags: [avro, schema-registry]
toc: true
image: /assets/img/cytonn-photography-GJao3ZTX9gU-unsplash.jpg
---

Avro é um formato de dados importante no mundo da computação distribuída, com
ele a quantidade de bytes trafegada pela rede será bem menor, se comparadada
com seu equivalente definido com JSON.

Ele também possui um poderoso mecanismo que permite a evolução dos esquemas
sem maiores dores-de-cabeça.

---

Neste artigo você entenderá como Avro dá suporte a evolução de esquemas, que
aliado ao [Schema Registry](https://docs.confluent.io/current/schema-registry/)
torna o trabalho muito mais simples.

---

## Avro

Avro é uma especificação para formato de dados mantida pelo
[grupo Apache](https://avro.apache.org/docs/current/spec.html). Ele tem muitas
utilidades e a principal e mais utilizada com Apache Kafka® é a
[Schema Resolution](https://avro.apache.org/docs/current/spec.html#Schema+Resolution).

> Acesse os equemas no projeto [esquemas-avro](https://github.com/kafkabr/esquemas-avro)

A resolução de esquema proporciona muita flexibilidade quando se identifica a
necessidade de incluir um novo campo, criando uma evolução por exemplo. Veja:

**Versão 1**:

```json
{
  "name":"ForwardDebitoExecutadoV1",
  "namespace":"com.kafkabr.e5o",
  "type":"record",
  "fields":[
    {
      "name":"valor",
      "type":"double"
    },
    {
      "name":"conta",
      "type":"string"
    },
    {
      "name":"descricao",
      "type":"string"
    },
    {
      "name":"apagar_v2",
      "type":"string",
      "default":"-apagado_v2-",
      "doc":"Campo que será eliminado na próxima versão"
    }
  ]
}
```

**Versão 2** (evolução):

```json
{
  "name":"ForwardDebitoExecutadoV2",
  "namespace":"com.kafkabr.e5o",
  "type":"record",
  "fields":[
    {
      "name":"valor",
      "type":"double"
    },
    {
      "name":"conta",
      "type":"string"
    },
    {
      "name":"descricao",
      "type":"string"
    },
    {
      "name":"metadados",
      "type":"string",
      "doc":"Novo campo requerido"
    }
  ]
}
```

Nos exemplos anteriores, na Versão 2 foi incluído um campo chamado `metadados`,
ele é requerido porque não define um valor padrão. Já a Versão 1 definiu um
valor padrão para o campo `apagar_v2`, tornando-o opcional.

Isso garante que eventos escritos (serializados) usando a Versão 2 do esquema
podem ser lidos (desserializados) com sua Versão 1. E este é um exemplo
clássico de compatibilidade `FORWARD`.

Então isso leva a outro item importante que é a compatibilidade entre
versões, ou seja, uma estratégia de evolução definida no inicio do
ciclo de vida do esquema.

## Evolução e Compatibilidade

Agora entra em cena o
[Schema Registry](https://docs.confluent.io/current/schema-registry/). Ele
gerencia a evolução e compatibilidade entre versões de esquemas, pois
ao registrar um novo esquema é obrigatório estabelecer uma estratégia
que será utilizada durante seu ciclo de vida.

As estratégias são:

> As estratégias de compatibilidade são relativas a nova versão do esquema

- `BACKWARD`: novo esquema pode ser utilizado para ler a versão
imediatamente anterior
- `BACKWARD_TRANSITIVE`: novo esquema pode ser utilizado para ler qualquer
versão anterior
- `FORWARD`: uma versão imediatamente anterior é capaz de ler eventos
escritos (serializados) com o novo esquema
- `FORWARD_TRANSITIVE`: qualquer versão anterior é capaz de ler eventos
escritos (serializados) com o novo esquema
- `FULL`: esquema é `BACKWARD` e `FORWARD` ao mesmo tempo
- `FULL_TRANSITIVE`: esquema é `FORWARD_TRANSITIVE` e `BACKWARD_TRANSITIVE`
ao mesmo tempo

## Interagir com Schema Registry

O modelo evolutivo do esquema pode ser definido ao registrá-lo no Schema Registry
ou herdado do padrão global, que aqui será demonstrado através de sua
[API Rest](https://docs.confluent.io/current/schema-registry/develop/api.html#schemaregistry-api).

> Substituir `http://localhost:8081` pela URL do registry

### Consultar estratégia evolutiva globlal no Schema Registry

```bash
curl -X GET http://localhost:8081/config
```

Saída. (Padrão é `BACKWARD`)

```json
{"compatibilityLevel":"BACKWARD"}
```

### Registrar esquema Versão 1 para um novo tópico

```bash
curl -X POST \
    -H "Content-Type: application/vnd.schemaregistry.v1+json" \
    --data '{"schema": "{\"name\":\"ForwardDebitoExecutadoV1\",\"namespace\":\"com.kafkabr.e5o\",\"type\":\"record\",\"fields\":[{\"name\":\"valor\",\"type\":\"double\"},{\"name\":\"conta\",\"type\":\"string\"},{\"name\":\"descricao\",\"type\":\"string\"},{\"name\":\"apagar_v2\",\"type\":\"string\",\"default\":\"-apagado_v2-\",\"doc\":\"Campo que ser\u00E1 eliminado na pr\u00F3xima vers\u00E3o\"}]}"}' \
    http://localhost:8081/subjects/conta-corrente-transacoes-value/versions
```

### Configurar a estratégia de evolução

> `FORWARD`

```bash
curl -X PUT \
     -H 'Content-Type: application/json' \
     --data '{"compatibility": "FORWARD"}' \
    http://localhost:8081/config/conta-corrente-transacoes-value
```

### Testar compatibilidade da evolução

> Se o teste for realizado antes da estratégia, então o resultado será
incompatível.

```bash
curl -X POST \
     -H "Content-Type: application/vnd.schemaregistry.v1+json" \
     --data '{"schema": "{\"name\":\"ForwardDebitoExecutadoV2\",\"namespace\":\"com.kafkabr.e5o\",\"type\":\"record\",\"fields\":[{\"name\":\"valor\",\"type\":\"double\"},{\"name\":\"conta\",\"type\":\"string\"},{\"name\":\"descricao\",\"type\":\"string\"},{\"name\":\"metadados\",\"type\":\"string\",\"doc\":\"Novo campo requerido\"}]}"}' \
     http://localhost:8081/compatibility/subjects/conta-corrente-transacoes-value/versions/1
```

Saída sobre a verificação de compatibilidade: 

```json
{"is_compatible":true}
```

### Registrar evolução, a Versão 2

> Esquema segue a estratégia `FORWARD`

```bash
curl -X POST \
     -H "Content-Type: application/vnd.schemaregistry.v1+json" \
     --data '{"schema": "{\"name\":\"ForwardDebitoExecutadoV2\",\"namespace\":\"com.kafkabr.e5o\",\"type\":\"record\",\"fields\":[{\"name\":\"valor\",\"type\":\"double\"},{\"name\":\"conta\",\"type\":\"string\"},{\"name\":\"descricao\",\"type\":\"string\"},{\"name\":\"metadados\",\"type\":\"string\",\"doc\":\"Novo campo requerido\"}]}"}' \
     http://localhost:8081/subjects/conta-corrente-transacoes-value/versions
```

### Registrar evolução incompatível

Quando se tenta registrar uma versão imcompatível com a estratégia evolutiva definida, um erro como este será retornado e o esquema rejeitado.

```json
{
  "error_code":409,
  "message":"Schema being registered is incompatible with an earlier schema for subject \"conta-corrente-transacoes-value\""
}
```

> Todos os esquemas estão no Github: [https://github.com/kafkabr/esquemas-avro](https://github.com/kafkabr/esquemas-avro)

_Obrigado e até o próximo!_

---
<span>Photo by <a href="https://unsplash.com/@cytonn_photography?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Cytonn Photography</a> on <a href="https://unsplash.com/s/photos/contract?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>