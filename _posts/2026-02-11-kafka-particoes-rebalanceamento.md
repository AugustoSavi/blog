---
title: "Kafka Internals: Partições, Grupos de Consumidores e Rebalanceamento"
date: 2026-02-11 09:00:00 -0300
categories: [Kafka, Sistemas Distribuídos]
tags: [kafka, sistemas distribuídos]
render_with_liquid: false
---

Você já se perguntou como o Kafka consegue processar milhões de mensagens por segundo enquanto o RabbitMQ sofre? O segredo está na forma como ele organiza os dados: as **Partições**.

## Divisão para Conquistar

Imagine um Tópico do Kafka como uma rodovia. Se houver apenas uma faixa, todos os carros (mensagens) ficam presos atrás um do outro. Se você criar 10 faixas (Partições), 10 carros podem andar lado a lado.

## Como as Partições funcionam

1.  **Escalabilidade:** Cada partição pode estar em um servidor diferente (Broker).
2.  **Ordem:** O Kafka garante a ordem das mensagens **apenas dentro de uma mesma partição**. Se a ordem global for vital, você terá problemas com múltiplas partições (ou precisará de uma chave de partição inteligente).
3.  **Paralelismo:** O número de partições define o seu limite máximo de consumidores paralelos.

## Grupos de Consumidores (Consumer Groups)

Se você tem um tópico com 4 partições e um grupo de consumidores com 2 instâncias, cada instância lerá de 2 partições. Se você subir mais 2 instâncias (total de 4), cada uma lerá de 1 partição.
**Mas atenção:** Se você subir uma 5ª instância, ela ficará **ociosa**. Você não pode ter mais consumidores em um grupo do que partições no tópico.

## Funcionamento Interno: O Protocolo de Replicação

Para garantir que os dados não se percam, o Kafka utiliza um modelo de replicação baseado em **Líder (Leader)** e **Seguidores (Followers)**.

1.  **O Papel do Controller:** Um dos brokers do cluster é eleito como o Controller. Ele é o cérebro do cluster, responsável por gerenciar estados de partições, tópicos e realizar a eleição de novos líderes caso um broker caia.
2.  **ISR (In-Sync Replicas):** Nem todo seguidor é igual. O ISR é o conjunto de réplicas que estão "em dia" com o líder. Se o líder cair, apenas uma réplica do ISR pode assumir a liderança para evitar perda de dados.
3.  **High Watermark:** É o offset da última mensagem que foi replicada com sucesso para todas as réplicas do ISR. O consumidor só consegue ler mensagens até este ponto, garantindo que ele nunca leia algo que possa "desaparecer" se o líder falhar.

## Zookeeper e KRaft

Historicamente, o Kafka dependia do Apache ZooKeeper para armazenar metadados e gerenciar o Controller. Isso criava um gargalo de escalabilidade e complexidade operacional (o famoso "metadados sobre metadados").

Com a migração para o **KRaft (Kafka Raft Metadata Mode)**, o Kafka passa a gerenciar seus próprios metadados internamente em um tópico especial. 
- **Vantagem:** O cluster escala para milhões de partições de forma muito mais eficiente.
- **Simplificação:** Menos um componente móvel para monitorar e manter na sua infraestrutura.

---

## O Temido Rebalanceamento (Rebalance)

Sempre que um consumidor entra ou sai do grupo (ou cai), o Kafka redistribui as partições entre os sobreviventes.
- **Problema:** Durante o rebalanceamento, o processamento pode parar (STW - Stop the World).
- **A Causa:** Geralmente consumidores lentos que demoram mais para processar do que o `max.poll.interval.ms`, fazendo o Kafka achar que eles morreram.

## Dica: Chaves de Partição (Partition Keys)

Se você quer que todas as mensagens do "Usuário A" sejam processadas na ordem correta, use o `userId` como chave da mensagem. O Kafka garante que a mesma chave sempre caia na mesma partição.

## Conclusão: A analogia do restaurante

Pense no Kafka como um restaurante de alta performance: as **partições** são as mesas, e os **consumidores** são os garçons. Se você tem 10 mesas e apenas 1 garçom, ele ficará sobrecarregado. Se você contratar 10 garçons, cada um atende uma mesa com perfeição. Mas se contratar 11 garçons, o último ficará parado na cozinha, pois não há mesa disponível para ele atender. Planeje suas partições pensando no número máximo de "garçons" que você precisará no horário de pico.