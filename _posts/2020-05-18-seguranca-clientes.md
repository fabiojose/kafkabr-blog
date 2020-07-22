---
title: Segurança no Produtor e no Consumidor
author: fabiojose
date:   2020-05-18 11:03:36 -0300
categories: [Exemplos de Código, Segurança]
tags: [snippet, configuração, produtor, consumidor, segurança, tls, ssl, jks, java]
toc: false
---

Segurança é fundamental em sistemas distribuídos e no #kafka não seria diferente. Existem muitas opções para proteção dos dados e autorização para Consumers, Producers e entre Brokers. E esta é uma configuração #java para Producer e Consumer com TLS.

```properties
# Tipo da segurança para TLS
security.protocol=SSL

# CA pública que assinou o certificado do Broker
ssl.truststore.location=/etc/securit/client.truststore.jks
ssl.truststore.password=test1234

# armazena o certificado privado do cliente
ssl.keystore.location=/etc/security/client.keystore.jks
ssl.keystore.password=test1234

# senha do certificado privado
ssl.key.password=CertTest1234
```