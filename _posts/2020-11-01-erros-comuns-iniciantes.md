---
title: 4 erros comuns ao trabalhar com Apache Kafka® e como evitá-los
author: fabiojose
date: 2020-11-01 05:50:37 -0300
categories: [Artigos, Avaliação]
tags: [iniciantes, tópico, partição, erros, offset, consumer, grupo-consumo, group.id]
toc: true
image: /assets/img/roman-mager-5mZ_M06Fc9g-unsplash.jpg
---

Toda pessoa iniciante no Apache Kafka® talvez tenha cometido pelo menos um
dos quatro erros que serão abordados aqui. O que natural e faz parte do
processo aprendizagem.

Veremos detalhes sobre como evitá-los e o que cada um deles pode
acarretar caso não se tome consciência daquilo que está em curso.

## Criar o primeiro tópico com apenas uma partição

O tópico criado automaticamente por um broker na configuração padrão terá
apenas uma partição. Esta é a situação mais comum quando se é uma pessoa iniciante no Apache Kafka®. Tópicos com apenas uma partição tem o comportamento similar ao
de [fila](https://pt.wikipedia.org/wiki/FIFO), que leva ao entendimento
incorreto de que o Kafka se trata de uma solução para fila. Algo que ele não é!

Tendo em mente que kafka é uma "fila", virtualmente imagina-se que para elavar
a taxa de processamento dos eventos basta criar mais partições. É ai que vem 
a surpresa ou frustação, dependendo do problema que se busca resolver.

Ao criar mais partições no tópico a ordenação global, garantida por uma solução
de filas, não se mantém. E isso pode acarretar uma grande dor-de-cabeça para o 
caso-de-uso em questão, porque no Apache Kafka® só existe
[manutenção da ordem](https://blog.kafkabr.com/posts/garantia-de-ordem) em
nível de partição.

Para evitar essa situação logo no primeiro momento de uso, crie seu tópico com 
duas ou mais partições. Assim você já terá contato com o modo real de operação
dos tópicos.

Exemplo do comando para criar `meu-topico` com `7` partições:

```bash
kafka-topics.sh --create \
  --bootstrap-server 'localhost:9092' \
  --replication-factor 1 \
  --partitions 7 \
  --topic 'meu-topico'
```

## Iniciar o consumer com as configurações padrão

Logo após subir uma instância do Apache Kafka® e produzir alguns eventos, vem
o segundo passo: consumi-los.

Então, um consumer é iniciado com as configurações requeridas:

- `bootstrap.servers`
- `group.id`
- `key.deserializer`
- `value.deserializer`

E neste momento vem uma surpresa, porque nenhum evento será consumido, a menos
que o producer esteja enviado continuamente. Mas como a idéia inicial foi
de que Kafka era uma fila, é bem provável que o producer tenha sido iniciado
para criar os eventos de desligado logo em seguida.

Parece estranho ou talvez até errado, mas é exatamente assim que um consumer
se comporta quando iniciado com suas configurações no valor padrão. Portando
evite isto incluindo a seguinte configuração:

```properties
auto.offset.reset=earliest
```

A estratégia `earliest` definida na configuração `auto.offset.reset`
garante que o consumer receberá todos os eventos do tópico em questão
desde de inicio. Ou seja, a partir do _offset_ mais antigo.

Assim evita-se a sensação de que o consumer não está "consumindo" ou de que 
não há eventos no tópico.

## Não confirmar o offset processado

Depois de consumir os eventos e ver que tudo funcionou, o consumer
é finalizado e segue-se com os estudos. Então ele é novamente iniciado
e todos os eventos que já haviam sido consumidos ou processados poderão
ser recebidos novamente.

Naturalmente isso leva a uma confusão: _porque está consumindo eventos
que já foram consumidos anteriormente?_

Mais uma vez o entendimento incorreto da fila pode estar causando uma
intepretação inconsistente sobre como Apache Kafka® trabalha na manutenção
dos eventos dentro de um tópico. Talvez se imagine que o evento consumido 
será removido do tópico apenas pela razão do seu consumo.

Mas não, eventos consumidos não são removidos dos tópicos, assim como uma 
fila tradicional o faz. Na verdade os eventos permanessem no mínimo 7 dias
no tópico (na configuração padrão). Depois disso são elegíveis para limpeza
mesmo que não tenham sido consumidos.

Bem, para evitar isso o consumer em sua configuração padrão já pode ajudar a
evitar isso, mas ajustá-la trará benefícios no funcionamento geral do consumo
e confirmação de _offset_.

```properties
# configuração padrão no consumer para commit automático
enable.auto.commit=true
```

O commit automático dá apoio à confirmação do _offset_ processado e o fará
perfeitamente para este momento inicial, onde se entende como Kafka funciona
no consumo de eventos. Mas poderá acarretar outro problema, então a
configuração mais utilizada é desligá-lo e trabalhar com a confirmação
programática.

```properties
# commit automático desligado
enable.auto.commit=false
```

```java
try(KafkaConsumer<String, String> consumer = 
      new KafkaConsumer<>(criarProperties())){
    
    consumer.subscribe("topico");

    ConsumerRecords<String, String> records = 
        consumer.poll(Duration.ofMillis(1000));

    // . . . processar os eventos

    // Confirmação programática e síncrona
    consumer.commitSync();
}
```

Neste exemplo de código Java a confirmação do _offset_ é programática e síncrona,
feita somente após o processamento dos eventos consumidos. 

## Ignorar a relação entre grupo de consumo e partições

A relação entre o grupo de consumo e partições é muito importante, compreendê-la
já no primeiro contato com o Apache Kafka® é essencial.

Mas na esmagadora maioria das vezes ela é ignorada, portando não é tratada com
a devida importância. E infelizmente ela se mostra em momentos
inoportunos, como quando a aplicação consumidora segue para produção.

Esta relação deve ser considerada desde o primeiro momento em que se idealiza
o uso de um tópico no Kafka para conectar produtores e consumidores.
Se o tópico tem trinta partições por exemplo, isso tem impacto em como será o
consumo. Mas em nada afeta como será a produção de eventos.

Entenda:

![Grupo de Consumo](/assets/img/particoes-grupo-de-consumo.png)

O que está ilustrado na imagem acima é que dado um grupo de consumo, o
número máximo de consumidores que poderão consumir neste grupo será 
igual ao número de partições disponíveis no tópico em questão. Em outras
palavras: mesmo que grupo de consumo contenha mais consumidores 
que partições, estes excedentes ficarão _idle_ e não consumem eventos.

Outra situação muito comum é pensar exatamente o contrário da afirmação acima,
ou seja, é pensar que quanto mais consumidores participarem do grupo de consumo
maior será a taxa de transferência. Imaginando que haverão mais deles
por partição.

Também pode-se imaginar que criando outro grupo de consumo irá resolver a
limitação imposta pelo número de partição. Mas isso causará problemas, porque
este novo grupo consumirá exatamente os mesmo eventos que o anterior.
Processamento em duplicidade.

---
<span>Photo by <a href="https://unsplash.com/@roman_lazygeek?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Roman Mager</a> on <a href="https://unsplash.com/s/photos/learning?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>
