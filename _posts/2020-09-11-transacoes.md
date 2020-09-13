---
title: Transação no Apache Kafka®
author: fabiojose
date:   2020-09-13 07:13:37 -0300
categories: [Artigos, Transações]
tags: [produtor, consumidor, transação]
toc: false
image: /assets/img/cytonn-photography-n95VMLxqM2I-unsplash.jpg
---

A partir da versão 0.11.0,  Kafka introduz transações na API do Produtor, garantido que exatamente todos os eventos serão produzidos ou exatamente nenhum em caso de erro.

Em linhas gerais as transações são indicadas para casos de uso onde serão produzidos eventos em diversos tópicos e partições diferentes, mas que fazem parte do mesmo contexto de transacional. Assim existirá a garantia de que exatamente todos serão persistidos no Kafka em caso de sucesso, ou exatamente nenhum, em caso de erro.

> Neste artigo serão descritos detalhes sobre as transações, exemplos de código Java, configurações e como ficam as transações em nossos serviços.

Nenhuma configuração especial será necessária no _broker_, pois o coordenador de transações está lá pronto e aguardando algum Producer iniciar a produção transacional. Mas elas são possíveis desde que a biblioteca cliente Kafka aí na sua linguagem tenha suporte. Estas são algumas implemetações que a suportam:

- Java
- Librdkafka: utilizada pelas bibliotecas nodejs, .net e golang

Outro ponto importante é que para trabalhar com transações o cluster Kafka deverá possuir
no mínimo 3 brokers. Isso vai garantir que haverá quorum para consenso no processamento
das transações.

Por fim, estaremos trabalhando com a garantia de entrega exatamente-uma-vez ou
_exactly-once_ em inglês. Ela é muito importante quando o caso de uso não
permite duplicatas ou perda de eventos.

## Configuração

A configuração é simples:

```properties
# Identificador único da transação
transactional.id=meu_id_unico_por_instância_de_produtor

# Timeout para finalização da transação
transaction.timeout.ms=60000

# Produção idempotente
acks=all
enable.idempotence=true
```

O valor para `transactional.id` deve ser globalmente único, ou seja, nenhum outro
produtor deverá utilizar o mesmo. Se isto acontecer àquele que estiver com transação
em andamento receberá um erro ao confirmá-la ou abortá-la.

- [ProducerFencedException](https://kafka.apache.org/24/javadoc/?org/apache/kafka/common/errors/ProducerFencedException.html): indica que outro produtor iniciou transação com o mesmo identificador.

## Usando Transações

> Código-fonte completo: https://github.com/kafkabr/transacional-java

Depois de configurar o produtor, agora é hora de colocar em prática.

Java é a linguagem escolhida para a implementação de referência dos clientes Kafka, já
outras linguagens são implementações compatíveis. Então, você poderá encontrará pequenas
diferenças no uso.

Crie o produtor normalmente, incluindo as [configurações](#configuração):

```java
try(KafkaProducer<String, String> producer = 
        new KafkaProducer<>(criarProducerConfigs())){

}
```

Informe ao coordenador de transações que você iniciará uma transação em breve:

```java
    // #########
    // Iniciar contexto transacional
    producer.initTransactions();
```

Inicie a transação:

```java
    // #####
    // Iniciar uma transação
    producer.beginTransaction();
```

Depois de produzir todos os eventos que fazem parte do mesmo contexto de negócio, basta confirmar a transação:

```java
    // #########
    // Confirmar uma transação 
    producer.commitTransaction();
```

O código-fonte completo fica assim:

```java
try(KafkaProducer<String, String> producer =
        new KafkaProducer<>(criarProducerConfigs())){

    // #########
    // Iniciar contexto transacional
    producer.initTransactions();

    // #####
    // Iniciar uma transação
    producer.beginTransaction();

    ProducerRecord<String, String> produce =
        new ProducerRecord<>(topicoProducer, r.value());
    
    // #####
    // Produzir registro transacional
    producer.send(produce);

    // Produzir mais eventos ...

    // #########
    // Confirmar contexto transacional
    producer.commitTransaction();
}
```

> Lembra muito a transação em um banco de dados relacional, não!?

__Tratamento de Erros__

Se durante a produção dos eventos alguma exceção for lançada, será necessário abortar a transação.

```java
try{
    // ...
}catch(ProducerFencedException | OutOfOrderSequenceException | AuthorizationException e){

    /*
     * Exceções lançadas antes do contexto transacional ser iniciado no broker,
     * portanto não existe uma transação para abortar.
     */

}catch(Exception e) {
    // #####
    // Abortar transação
    producer.abortTransaction();
}
```

Acesse o código fonte completo no Github, ele está completo e você utilizá-lo como 
base para seus estudos e projetos.

- https://github.com/kafkabr/transacional-java 

## E os Consumidores?

Não existe na [Consumer API](https://kafka.apache.org/documentation/#consumerconfigs) uma
definição para consumo transacional de eventos, porém, é possível se beneficiar das transações no produtor.

__Configurações:__

```properties
# Não consome eventos com transação em voô
isolation.level=read_committed
```

Depois de consumir eventos e processá-los, sempre é necessário confirmar (_commit_) o
maior _offset_ recebido. E esta ação é uma produção de evento por que o gerencimento
de _offsets_ é feito pelo broker em um tópico interno chamado `__consumer_offsets`.

Então, é possível realizar o consumor transacioal sem problemas. Veja:

```java
ConsumerRecords<String, String> records =
    consumer.poll(Duration.ofSeconds(5));

// Iniciar uma transação
producer.beginTransaction();

// Consumir eventos . . .

// Processar eventos consumidos . . . 

//###### Aqui está o commit dos offsets
producer.sendOffsetsToTransaction(
    offsetsProcessados(records),
    "group.id do consumidor");

// Confirmar contexto transacional
producer.commitTransaction();
```

> No código fonte exemplo você encontra a implementação do método
`offsetsProcessados()`

Pouco antes de invocar `producer.commitTransaction()` o método `producer.sendOffsetsToTransaction()` é executado. Através dele que se realiza
o `commit` dos _offsets_ consumidos, sendo proibitivo invocar `consumer.commitSync()`, que levaria a confirmá-los sem participar da transação.

A partir deste momento o consumo será confirmado somente se a transação for confirmada,
ou seja, tudo-ou-nada, tanto para o consumo, quanto para a produção dos eventos.

## E as Outras Transações?

E as transações locais? Aquelas para bancos de dados relacionais com JTA,
por exemplo.

Transações Apache Kafka não tem relação ou compatibilidade com qualquer outra transação
em andamento nas aplicações que consumem ou produzem eventos.

Vamos a um pseudo-código com transação JDBC em Java:

```java

// Transação JDBC
jdbc.setAutoCommit(false);

try(KafkaProducer<String, String> producer =
        new KafkaProducer<>(criarProducerConfigs())){

    producer.initTransactions();
    producer.beginTransaction();

    ProducerRecord<String, String> produce =
        new ProducerRecord<>(topicoProducer, r.value());
    
    producer.send(produce);

    // Operações JDBC
    jdbc.execute("INSERT INTO ...");
    jdbc.execute("DELETE FROM ...");

    producer.commitTransaction();

    // Confirmar transação JDBC
    jdbc.commit();
}
```

Pouco antes de invocar o `connection.commit()`, a transação Apache Kafka também foi
confirmada. Não existe qualque relação entre elas e se o _commit_
JDBC falhar, os eventos produzidos e/ou consumidos já teram sua transação confirmada
e não haverá qualquer possibilidade de __rollback__.

> Transações locais (JDBC, JTA) e transações Apache Kafka não são atômicas entre si.

O que leva a pensar:

- Qual _commit_ realizar primeiro?

A resposta é simples: qualquer um deles que for confirmado primeiro poderá levar
a um estado inconsistente, se houver necessidade de consistência mútua.

Então se produzir e/ou consumir e persistência devem pertencer ao mesmo contexto
transacional será necessário uma implementação diferente, robusta e recheada de
truques. Mas este será o assunto para o próximo artigo, até lá!

---
<span>Photo by <a href="https://unsplash.com/@cytonn_photography?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Cytonn Photography</a> on <a href="https://unsplash.com/s/photos/transaction?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>
