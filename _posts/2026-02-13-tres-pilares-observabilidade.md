---
title: "Os 3 Pilares da Observabilidade: Além do \"Está no ar?\""
date: 2026-02-13 09:00:00 -0300
categories: [DevOps, Observabilidade]
tags: [devops, observabilidade]
render_with_liquid: false
---

Antigamente, monitorar um sistema era saber se o servidor estava ligado. Em um mundo de microserviços distribuídos na nuvem, "estar ligado" não é suficiente. Você precisa de **Observabilidade**. Mas o que compõe esse conceito? Vamos falar sobre os **Três Pilares**.

## O Detetive de Sistemas

Observabilidade é a capacidade de entender o estado interno de um sistema apenas olhando para os seus dados externos. É a diferença entre saber que o carro parou e saber que ele parou porque a vela de ignição falhou no cilindro 3.

## 1. Métricas (O "Quanto?")
São dados agregados ao longo do tempo (CPU, Memória, Requisições por Segundo, Latência).
- **Ferramentas:** Prometheus, Grafana, Datadog.
- **Uso:** Criar dashboards e alertas. "A latência aumentou 20% nos últimos 5 minutos".

## 2. Logs (O "O quê?")
São registros de eventos específicos que aconteceram no sistema.
- **Ferramentas:** ELK Stack (Elasticsearch, Logstash, Kibana), Splunk, Graylog.
- **Uso:** Investigar a causa raiz de um erro. "O usuário 123 recebeu um NullPointerException na linha 45".
- **Dica:** Use **Logs Estruturados** (JSON) para facilitar a busca.

## 3. Traces (O "Onde?")
Mostram o caminho de uma única requisição através de todos os microserviços.
- **Ferramentas:** Jaeger, Zipkin, AWS X-Ray.
- **Uso:** Identificar qual serviço na cadeia está causando lentidão. "A requisição demorou 2 segundos porque o Serviço de Inventário demorou 1.8s".

## Contexto de Uso (ex: Datadog)

Muitos times utilizam ferramentas modernas como o **Datadog**, que unifica esses três pilares em uma única plataforma. Isso permite que você clique em um erro no log e veja instantaneamente o trace daquela requisição e a saúde do servidor naquele exato momento.

## Curiosidade: Cardinalidade

Ao criar métricas, cuidado com a **Cardinalidade**. Tentar criar uma métrica para cada `userId` vai explodir o seu banco de métricas e custar uma fortuna. Guarde dados de alta cardinalidade (como IDs) nos Logs e Traces, e dados agregados nas Métricas.

## Uma reflexão final

Da próxima vez que seu sistema apresentar uma lentidão intermitente em produção, pergunte-se: eu tenho os dados necessários para diagnosticar isso em 5 minutos ou vou precisar de 5 horas injetando novos logs e fazendo novos deploys? A observabilidade não é sobre o que você monitora hoje, mas sobre quais perguntas você será capaz de responder amanhã sem precisar alterar uma única linha de código.