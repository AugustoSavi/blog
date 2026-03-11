---
title: "Garantias de Entrega no Kafka: At-least-once ou At-most-once?"
date: 2026-02-10 09:00:00 -0300
categories: [Kafka, Mensageria]
tags: [kafka, mensageria]
render_with_liquid: false
---

Em sistemas de mensageria, o que acontece quando uma rede falha no meio do envio? A mensagem se perde? Ela é duplicada? Entender as **Garantias de Entrega** é vital para não cobrar o cliente duas vezes ou, pior, esquecer de cobrar.

## O Trilema do Produtor

Existem três modelos principais de garantia de entrega no Kafka, controlados principalmente pela configuração de **Acks (Acknowledgments)** e **Offsets**.

## 1. At-most-once (No máximo uma vez)
A mensagem é enviada e o produtor não espera confirmação. Se o broker cair, a mensagem se foi para sempre.
- **Config:** `acks=0`.
- **Uso:** Logs não críticos ou métricas de telemetria onde perder um dado não é o fim do mundo.

## 2. At-least-once (No mínimo uma vez)
O produtor espera o "OK" do broker. Se não receber, ele tenta de novo. Isso garante que a mensagem chegue, mas se o "OK" se perder na rede, o produtor pode enviar a mesma mensagem duas vezes.
- **Config:** `acks=1` ou `acks=all`.
- **Uso:** A maioria dos sistemas. **Exige que o consumidor seja Idempotente.**

## 3. Exactly-once (Exatamente uma vez)
O "Santo Graal". Garante que a mensagem chegue e seja processada apenas uma vez, mesmo com falhas e retentativas.
- **Como:** Usa transações no Kafka e produtores idempotentes (`enable.idempotence=true`).
- **Uso:** Sistemas financeiros críticos. É mais pesado e exige configurações precisas tanto no produtor quanto no consumidor.

## Por que At-least-once é o padrão?

Porque é o equilíbrio perfeito entre performance e segurança. No entanto, ele coloca a responsabilidade no **Consumidor**. Se o seu consumidor recebe uma mensagem de "Cobrar R$ 50", ele deve verificar se aquele ID de transação já foi processado antes de agir.

## Curiosidade: O custo do 'acks=all'

Quando você usa `acks=all`, o Kafka espera que **todas** as réplicas sincronizadas confirmem o recebimento. Isso traz uma segurança incrível contra perda de dados, mas aumenta a latência da sua aplicação.

## Conclusão

Escolher a garantia de entrega errada pode gerar inconsistências silenciosas no seu sistema. Saiba quando você pode perder dados e quando você deve estar preparado para lidar com duplicatas.