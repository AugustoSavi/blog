---
title: "Teorema CAP: A Escolha Impossível dos Sistemas Distribuídos"
date: 2026-02-07 09:00:00 -0300
categories: [Sistemas Distribuídos, Arquitetura]
tags: [sistemas distribuídos, arquitetura]
render_with_liquid: false
---

Você está desenhando um sistema crítico e precisa decidir: em caso de uma falha na rede, o sistema deve parar de responder ou deve responder dados potencialmente desatualizados? Essa é a essência do **Teorema CAP**.

## Você só pode escolher dois

O Teorema CAP diz que um sistema distribuído só pode garantir, simultaneamente, duas das três propriedades abaixo:
1.  **C (Consistency):** Todos os nós veem os mesmos dados ao mesmo tempo.
2.  **A (Availability):** Toda requisição recebe uma resposta (sucesso ou falha).
3.  **P (Partition Tolerance):** O sistema continua funcionando apesar de falhas de comunicação entre os nós.

## O Grande Segredo: P não é opcional

Em sistemas distribuídos na nuvem, falhas de rede (**Partitions**) **vão acontecer**. Portanto, a escolha real quase sempre é entre **Consistência (CP)** ou **Disponibilidade (AP)** durante uma falha.

- **CP (Consistency + Partition Tolerance):** Se a rede falha, o sistema para de responder para não entregar dados inconsistentes. (Ex: HBase, MongoDB em certas configs).
- **AP (Availability + Partition Tolerance):** Se a rede falha, o sistema continua respondendo, mas alguns nós podem ter dados antigos. (Ex: Cassandra, DynamoDB).

## Exemplo Prático: Saldo de Cartão

- Se você prioriza **Consistência (CP)**, um usuário pode ter o cartão negado porque o nó A não consegue falar com o nó B para confirmar o saldo. Segurança máxima, mas experiência de usuário ruim.
- Se você prioriza **Disponibilidade (AP)**, o usuário consegue passar o cartão, mas existe o risco dele gastar mais do que tem porque o nó que aprovou a compra ainda não sabia de uma compra feita segundos antes em outro nó.

## Curiosidade: O Teorema PACELC

O CAP só fala o que acontece quando há falha. O **PACELC** estende isso para o estado normal: "Se houver partição (P), escolha entre A e C. Senão (E), escolha entre Latência (L) e Consistência (C)".

## Uma Pergunta para o Próximo Incidente de Rede

Quando o seu monitoramento indicar uma partição de rede entre os seus data centers, qual será a sua prioridade: manter a integridade absoluta dos saldos dos clientes (Consistência) ou garantir que eles ainda consigam realizar compras emergenciais, mesmo com risco de saldo negativo temporário (Disponibilidade)? O Teorema CAP não é um problema a ser resolvido, mas uma realidade a ser aceita. Você já definiu com o seu Product Owner qual é o "preço da perfeição" para o seu negócio?