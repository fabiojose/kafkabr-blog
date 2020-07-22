---
title: Apagar Eventos do Apache Kafka? 
author: fabiojose
date:   2020-05-25 22:03:36 -0300
categories: [Artigos, Tópico]
tags: [tópico]
toc: false
---

Você já deve ter-se perguntado se é possível apagar eventos do Kafka, não? Bem, isso realmente não é possível.

Mas existe uma forma especial utilizada principalmente para atender o direito ao esquecimento, requerido pela LGPD em alguns países.

⚠️ Atenção! ⚠️ Entenda todos os impactos antes de alterar qualquer tópico que você já tenha ai no seu cluster Kafka.

---

Isso não se trata de um hack ou algo do tipo, até porque ele está descrito no livro ["Kafka: The Definitive Guide"](https://www.confluent.io/resources/kafka-the-definitive-guide) 😊.

---

Primeiro, seu tópico deverá ser compactado, ou seja, a configuração `cleanup.policy` deve ser igual a `compact`. Ele garantirá que somente o evento mais recente será mantido.

Em um tópico compactado é mandatório produzir eventos com chave de partição, o atributo `key`. É com base nele que o Kafka manterá o evento mais recente.

Digamos que existe o tópico `clientes` com os seguintes eventos na partição `3`:

```
CHAVE   VALOR DO EVENTO
C001    Cliente 1y
C002    Cliente 2+
C003    Cliente 3@
C001    Cliente 1π
```

Veja que a chave `C001` tem um evento mais recente, que será mantido após a compactação do tópico:

```
CHAVE   VALOR DO EVENTO
C002    Cliente 2+
C003    Cliente 3@
C001    Cliente 1π
```

Agora que você entendeu a compactação, para "apagar" um evento, o `C003` por exemplo, bastará produzir um evento sem valor com a mesma chave:

```
CHAVE   VALOR DO EVENTO
C001    Cliente 1y
C002    Cliente 2+
C003    Cliente 3@
C001    Cliente 1π
C003
```

Então, após compactação do tópico, o cliente `C003` deixará de existir:

```
CHAVE   VALOR DO EVENTO
C001    Cliente 1y
C002    Cliente 2+
C001    Cliente 1π
C003
```

Aqui foi um exemplo, mas a chave de partição poderia ser o CPF, CNPJ ou qualquer outra valor pertinente ao caso se uso.

---

Simples, não? Mas não se engane, porque empregar chave de partição não uma regra geral. Então, não mude as configurações do seu tópico antes de entender as consequências.

Se você já tem um tópico com eventos produzidos sem chave, a primeira coisa que acontecerá quando mudar a `cleanup.policy` para `compact` é que todos eles serão eliminados 😬. Mudar configurações de tópicos só com muita análise dos impactos.

---

É isso. 

Obrigado pelo leitura e até o próximo!
