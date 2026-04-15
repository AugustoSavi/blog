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

---

## Otimização: Throughput vs Latência

Configurar as garantias de entrega é apenas metade da batalha. A outra metade é decidir o quão rápido você quer que essas mensagens viajem. No Kafka, existe um trade-off clássico entre **vazão (throughput)** e **atraso (latency)**.

### Tuning do Produtor
- **`batch.size`:** Define o tamanho máximo (em bytes) de uma "sacola" de mensagens. Sacolas maiores aumentam o throughput (mais dados por requisição), mas aumentam a latência (esperar a sacola encher).
- **`linger.ms`:** É o tempo que o produtor espera por mais mensagens antes de enviar o batch atual. Se você definir `linger.ms=5`, o produtor esperará até 5ms para acumular mensagens. Isso reduz drasticamente o número de requests enviadas, melhorando a eficiência do broker às custas de 5ms de latência.
- **`compression.type`:** Usar `snappy` ou `lz4` reduz o tamanho dos dados na rede e no disco, aumentando o throughput, mas consome um pouco mais de CPU do produtor para comprimir.

### Tuning do Consumidor
- **`fetch.min.bytes`:** O consumidor só recebe dados se houver pelo menos X bytes disponíveis no broker. Aumentar isso economiza CPU e rede, mas aumenta a latência.
- **`fetch.max.wait.ms`:** Quanto tempo o consumidor espera pelo `fetch.min.bytes` antes de desistir e pegar o que tiver disponível.

---

## Por que At-least-once é o padrão?

Porque é o equilíbrio perfeito entre performance e segurança. No entanto, ele coloca a responsabilidade no **Consumidor**. Se o seu consumidor recebe uma mensagem de "Cobrar R$ 50", ele deve verificar se aquele ID de transação já foi processado antes de agir.

## Curiosidade: O custo do 'acks=all'

Quando você usa `acks=all`, o Kafka espera que **todas** as réplicas sincronizadas confirmem o recebimento. Isso traz uma segurança incrível contra perda de dados, mas aumenta a latência da sua aplicação.

## Takeaway prático

Para a maioria dos sistemas de mensageria, a configuração ideal segue a regra de ouro: utilize `acks=all` com `min.insync.replicas=2` no produtor para garantir durabilidade, e implemente lógica de **idempotência** no consumidor. Essa combinação oferece o melhor equilíbrio entre segurança contra perda de dados e resiliência a falhas de rede, sem a complexidade extrema das transações exatamente-uma-vez.