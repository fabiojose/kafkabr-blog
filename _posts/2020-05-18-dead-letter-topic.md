---
title: Como implementar Dead-letter Topic com Spring Kafka 
author: fabiojose
date:   2020-05-18 22:03:36 -0300
categories: [Artigos, Arquitetura]
tags: [tópico, spring, spring-kafka, dlt, java]
toc: true
---

_Dead-letter Topic_, [_Dead-letter Queue_](https://en.m.wikipedia.org/wiki/Dead_letter_queue) ou em bom e velho português: __Tópicos de mensagens não-entregues__. São tópicos necessários em sistemas distribuídos onde a comunicação é assíncrona e através de brokers como o Kafka.

Os dados que chegam nestes tópicos passaram por todas as tentativas possíveis para tratamento de erros e já não resta mais nada a ser feito, se não, a intervenção humana. Assim, __não será qualquer erro__ que levará mensagens ou eventos a serem publicados em um tópico _dead-letter_.

> No mundo Kafka um tópico dead-letter é destinado aos registros consumidos, que por algum erro irrecuperável não puderam ser processados com sucesso.

---

Neste artigo será demonstrada uma abordagem para implementar DLT no Kafka, utilizando Java e Spring. Para os impacientes 🧐, estes são os fontes :

- [Código fonte no GitHub](https://github/fabiojose/spring-kafka-dlt-ex)

---

### Classificar Erros

Antes de escrever qualquer linha de código é necessário classificar os erros e como serão tratados.

Existem dois tipos de erros: 

- recuperáveis
- não-recuperáveis

Os erros __recuperáveis__ ou que podem ser tratados, são aqueles onde  alguma abordagem será empregada para tentar finalizar o fluxo com sucesso.

Por exemplo, no Java, os erros [`ConnectException`](https://docs.oracle.com/javase/8/docs/api/java/net/ConnectException.html) e [`UnknownHostException`](https://docs.oracle.com/javase/8/docs/api/java/net/UnknownHostException.html) podem ser recuperados através de retentativas, pois normalmente são causados por instabilidades momentâneas no serviço ou na rede.

Já os __não-recuperáveis__ são aqueles que independente do que seja feito, não será possível finalizar o fluxo com sucesso. Logo, estes são candidatos a seguirem diretamente para o tópico dead-letter. E como exemplo, se você utiliza Avro, o erro [`AvroMissingFieldException`](http://avro.apache.org/docs/1.9.2/api/java/index.html) indica a ausência de um campo requerido, não havendo nada a fazer. Portanto será inútil a retentativa, por exemplo.

---

__E além dos erros técnicos, você também deverá classificar seus erros de negócio e escolher uma estratégia para tratá-los.__

---

Bem, agora veremos como implementar DLT para problemas técnicos no Java. E é bem provável que você encontre equivalentes na sua linguagem ou framework.

### Recuperáveis

Aqui segue uma lista de erros técnicos recuperáveis que, em nome da simplicidade, serão tratados através de retentativas.

- [`ConnectException`](https://docs.oracle.com/javase/8/docs/api/java/net/ConnectException.html)
- [`UnknownHostException`](https://docs.oracle.com/javase/8/docs/api/java/net/UnknownHostException.html)
- `ProductNotFoundException`: a título de exemplo, este erro de negócio foi definido como tratável. Porque um serviço chamado _Catalogo_, que faz [event-sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) das mudanças emitidas pelo serviço _Produto_, ainda não processou o evento com o produto em questão. Mas através da retentativa seu fluxo poderá finalizar com sucesso.

### Não-recuperáveis

Estes são alguns dos erros não-recuperáveis, ou seja, se passarem pelo mesmo processo de retentativas somente consumiriam recursos, sem chances de finalizar com sucesso.

- [`AvroMissingFieldException`](http://avro.apache.org/docs/1.9.2/api/java/index.html)
- [`NullPointerException`](https://docs.oracle.com/javase/8/docs/api/java/lang/NullPointerException.html)
- [`ClassCastException`](https://docs.oracle.com/javase/8/docs/api/java/lang/ClassCastException.html)
- [`RecordTooLargeException`](https://kafka.apache.org/20/javadoc/index.html?org/apache/kafka/common/errors/RecordTooLargeException.html)
- `StockNotAvailableException`: como exemplo, este erro de negócio foi classificado como irrecuperável. Porque a falta de produtos no estoque não será resolvida com retentativas, por exemplo.

### Implementação

> Existe um [ótimo artigo](https://eng.uber.com/reliable-reprocessing/) no blog de engenharia do Uber que descreve uma arquitetura para dead-letter. Vale a leitura!

Esta implementação foi dividida em três partes:

- tópicos
- construção
- tratamento

#### Tópicos

Para cada processo que terá tratamento _dead-letter_, será criado um tópico com o sufixo `-dlt`. Tome como exemplo um processo que realiza a reserva de estoque a partir das ordens de compra publicadas no tópico `ordem-compra`. Então, ao invés de criar um tópico ordem-compra-dlq, será utilizado um nome relevante ao processo que está falhando:

- `reservar-estoque-dlt`

E mais os tópicos para retentativas, que devem ser quantos forem necessários até finalmente o registro chegar ao tópico _dead-letter_. Digamos que serão no máximo quatro retentativas além da inicial:

- `reservar-estoque-retry-1`
- `reservar-estoque-retry-2`
- `reservar-estoque-retry-3`
- `reservar-estoque-retry-4`

Os tópicos devem ter características que não interfiram no andamento das retentativas ou na publicação do tópico dead-letter. Um exemplo é a configuração `max.message.bytes`, que pode acarretar um erro chamado `message too large`. Portanto os tópicos deverão permitir mensagens maiores, caso contrário também não será possível utilizá-los porque são idênticos ao principal, inclusive com as mesmas limitações.

#### Construção

A construção foi feita com Spring Kafka porque ele já vem com muitas utilidades para tratamento para _dead-letter_, o que contribui muito com a produtividade se ela é o foco no seu projeto. Mas nada impede que você escreva a mesma solução com Clientes Kafka no Java ou na sua linguagem do seu projeto.

Para utilizar os recursos destinados a dead-letter, é necessário customizar a fábrica de objetos encarregada de produzir _listeners_ Kafka no Spring.

Assim, a configuração programática foi sub-dividida em três partes:

- `resolver`: responsável por determinar o tópico destino do registro que está no contexto do erro.
- `errorHandler`: manipulador do erro
- `kafkaListernerContainerFactory`: fabrica instâncias que são utilizadas nos métodos anotados com `@KafkaListener`

E para cada uma das sub-divisões existe uma implementação __Main__ e outra __Retry__. Como é possível imaginar, uma cuida das configurações para o processamento principal outro para as retentativas.

__Main resolver__, responsável por determinar qual o tópico destino com base no erro-raiz lançado ao processar o registro consumido do tópico `ordem-compra`:

```java
@Bean
public BiFunction < ConsumerRecord < ? , ? > , Exception, TopicPartition >
 mainResolver() {

  return new BiFunction < ConsumerRecord < ? , ? > , Exception, TopicPartition > () {
   @Override
   public TopicPartition apply(ConsumerRecord < ? , ? > r, Exception e) {

    // ####
    // Por padrão, quando é não-recuperável, segue diretamente p/ dead-letter
    TopicPartition result =
     new TopicPartition(dltTopic,
      QUALQUER_PARTICAO);

    // ####
    // Trata-se de um erro recuperável?
    final boolean recuperavel = isRecuperavel(e);
    if (recuperavel) {

     Optional < String > origem =
      topicoOrigem(r.headers())
      .or(() -> Optional.of(NENHUM_CABECALHO));

     // ####
     // Se origem for outro tópico, segue para o primeiro retry
     String destino =
      origem
      .filter(topico -> !topico.matches(retryTopicsPattern))
      .map(t -> retryFirstTopic)
      .orElse(dltTopic);

     result = new TopicPartition(destino, QUALQUER_PARTICAO);
    }

    return result;
   }
  };
 }
```

__Main error handler__, que utiliza o __Main resolver__ e é responsável por definir duas configurações essenciais para tratamento dos erros:

- [`DeadLetterPublishingRecoverer`](https://docs.spring.io/spring-kafka/api/org/springframework/kafka/listener/DeadLetterPublishingRecoverer.html): inicia o fluxo DLT, que é delegado pelo SeekToCurrentErrorHandler caso as retentativas locais não resolvam o erro.
- [`SeekToCurrentErrorHandler`](https://docs.spring.io/spring-kafka/api/org/springframework/kafka/listener/SeekToCurrentErrorHandler.html): manipula qualquer erro que seja lançado no método anotado com `@KafkaListener`. Naturalmente, se você capturá-los com `catch` e não permitir que eles _subam na pilha_, não será possível tratá-los.

```java
@Bean
public SeekToCurrentErrorHandler mainErrorHandler(
 @Qualifier("mainResolver")
 BiFunction < ConsumerRecord < ? , ? > , Exception, TopicPartition > resolver,
 KafkaTemplate < ? , ? > template) {

 // ####
 // Recuperação usando dead-letter 
 DeadLetterPublishingRecoverer recoverer =
  new DeadLetterPublishingRecoverer(template, resolver);

 // ####
 // Tentar 3x localmente antes de iniciar o fluxo dead-letter
 SeekToCurrentErrorHandler handler =
  new SeekToCurrentErrorHandler(recoverer, RETENTAR_3X);

 // ####
 // Lista das exceções não-recuperáveis, para evitar o retry local
 excecoes.getNaoRecuperavies().forEach(e ->
  handler.addNotRetryableException(e));

 return handler;
}
```

__Main listener factory__, encarregado de fabricar os consumidores que processam registros do tópíco `ordem-compra`.

```java
@Bean
public KafkaListenerContainerFactory < ConcurrentMessageListenerContainer < String, GenericRecord >>
 mainKafkaListenerContainerFactory(
  @Qualifier("mainErrorHandler") SeekToCurrentErrorHandler errorHandler,
  KafkaProperties properties,
  ConsumerFactory < String, GenericRecord > factory) {

  ConcurrentKafkaListenerContainerFactory < String, GenericRecord > listener =
   new ConcurrentKafkaListenerContainerFactory < > ();

  listener.setConsumerFactory(factory);

  // ####
  // Utilizando o mainErrorHandler para tratar os erros
  listener.setErrorHandler(errorHandler);

  // Falhar, caso os tópicos não existam?
  listener.getContainerProperties()
   .setMissingTopicsFatal(missingTopicsFatal);

  // Commit do offset no registro, logo após processá-lo no listener
  listener.getContainerProperties().setAckMode(AckMode.MANUAL);

  // Commits síncronos
  listener.getContainerProperties().setSyncCommits(Boolean.TRUE);

  return listener;
 }
```

Então, basta anotar o método como segue.

Mas vale ressaltar que o offset sempre deverá ser confirmado, porque todo o fluxo que trata os erros será feito por retentativa e talvez culminando no tópico dead-letter. _Analise o seu caso-de-uso e entenda se esta abordagem também se aplicada._

```java
@KafkaListener(
 id = "main-kafka-listener",
 topics = "${app.kafka.consumer.topics}",
 containerFactory = "mainKafkaListenerContainerFactory"
)
public void consume(@Payload ConsumerRecord < String, GenericRecord > record,
 Acknowledgment ack) throws Exception {

 try {

  // #### 
  // Processar
  process(record);

 } finally {

  // ####
  // Sempre confirmar o offset
  ack.acknowledge();
 }

}
```

Já a configuração para o processamento das retentativas segue moldes similares, com algumas exceções:

- `resolver`: retorna qual o próximo tópico na sequência de retentativas ou se é o `reservar-estoque-dlt`, caso já tenham passadas por todas.
- `errorHandler`: nenhuma retentiva local.

Este é o método anotado com `@KafkaListener` que tratará os tópicos `reservar-estoque-retry`:

```java
@KafkaListener(
 id = "retry-kafka-listener",
 topicPattern = "${app.kafka.dlt.retry.topics.pattern}",
 containerFactory = "retryKafkaListenerContainerFactory",
 properties = {
  "fetch.min.bytes=${app.kafka.dlt.retry.min.bytes}",
  "fetch.max.wait.ms=${app.kafka.dlt.retry.max.wait.ms}"
 }
)
public void retry(@Payload ConsumerRecord < String, GenericRecord > record,
 Acknowledgment ack) throws Exception {

 try {

  // ####
  // Reprocessar
  process(record);

 } finally {
  // ####
  // Sempre confirmar o offset
  ack.acknowledge();
 }

}
```

- `topicPattern`: consumir todos os tópicos `reservar-estoque-retry`
- `fetch.min.bytes` e `fetch.max.wait.ms`: utilizados para provocar um certo atraso no consumo dos registros, visto que sem eles o consumo seria praticamente instantâneo.

Por fim, o `application.properties` será assim: 

```properties
app.kafka.dlt.retry.topics=4
app.kafka.dlt.retry.topics.pattern=reservar-estoque-retry-[0-9]+
app.kafka.dlt.retry.topic.first=reservar-estoque-retry-1
app.kafka.dlt.topic=reservar-estoque-dlt

# Lista de exceções recuperáveis
app.kafka.dlt.excecoes.recuperaveis[0]=java.net.ConnectException
app.kafka.dlt.excecoes.recuperaveis[1]=java.net.UnknownHostException

# Lista de exceções não-recuperáveis
app.kafka.dlt.excecoes.naoRecuperaveis[0]=org.apache.avro.AvroMissingFieldException
app.kafka.dlt.excecoes.naoRecuperaveis[1]=java.lang.NullPointerException

# Provocar atraso no processmento de retentativa
app.kafka.dlt.retry.max.wait.ms=20000
app.kafka.dlt.retry.min.bytes=52428800
```

Todo a implementação está disponível no Github, _clone it and have fun!_

- [Código fonte no GitHub](https://github/fabiojose/spring-kafka-dlt-ex)

---

Quando o registro atinge o último tópico de retentativa, que neste caso é o `reservar-estoque-4` e não seja processado com sucesso, finalmente ele será publicado no tópico _dead-letter_, então a equipe deverá estar preparada para o tratamento adequado.

#### Tratamento

Bem, neste momento o registro com problemas já desembarcou no tópico `reservar-estoque-dlt` e bem antes disso acontecer um sistema preciso de monitoramento dos tópicos deveria ter alertado sobre o uso dos tópicos para retentativas, principalmente se os registros estão atingindo o `reservar-estoque-retry-4`.

> Veja [neste artigo](https://medium.com/@alvarobacelar/monitorando-um-cluster-kafka-com-ferramentas-open-source-a4032836dc79) como monitorar seu cluster Kafka.

A maneira mais prudente de tratamento, além do monitoramento e um ótimo sistema de rastreamento distribuído, será construir um processo em que para cada registro publicado no DLT, tickets e notificações ChatOps deveram ser enviados para a equipe responsável.

> Também são necessárias ferramentas adequadas para, por exemplo, editar registros e colocá-los novamente no tópico original.

---

Bem, é isso! Até o próximo.

