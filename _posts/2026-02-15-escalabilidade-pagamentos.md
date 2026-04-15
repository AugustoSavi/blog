---
title: "Escalabilidade em Pagamentos: O Desafio do \"Dia do Benefício\""
date: 2026-02-15 09:00:00 -0300
categories: [System Design, Escalabilidade]
tags: [system design, escalabilidade]
render_with_liquid: false
---

Em sistemas de larga escala, o maior desafio técnico não é o dia a dia, mas sim os picos de tráfego. Imagine milhares de empresas creditando benefícios simultaneamente e milhões de usuários tentando usar seus cartões na hora do almoço. Como desenhar um sistema que não ajoelha sob essa carga?

## O Efeito Manada

Em sistemas financeiros, o tráfego é altamente previsível mas brutal. No dia 01 ou no dia 15, a carga pode ser 100x maior que a média. Se o seu sistema for desenhado de forma síncrona, ele vai travar.

## Estratégia 1: Desacoplamento com Mensageria

**Nunca processe o crédito de benefício de forma síncrona na requisição HTTP.**
- O cliente (RH da empresa) envia o arquivo/solicitação.
- O sistema valida o formato, salva o arquivo no S3 e coloca uma mensagem no Kafka: "Processar Lote X".
- O cliente recebe um `202 Accepted` imediatamente.
- Um exército de consumidores lê do Kafka e processa os créditos em background, respeitando o limite do banco de dados.

## Estratégia 2: Sharding e Particionamento

Se o seu banco MySQL central está sofrendo, você precisa dividir para conquistar.
- **Database Sharding:** Dividir os dados por `companyId`. As empresas A-M ficam no Banco 1, N-Z no Banco 2.
- **Read Replicas:** Use réplicas de leitura para consultas de saldo e extrato, deixando o banco principal (Master) focado apenas em escritas (créditos e débitos).

## Estratégia 3: Cache de Saldo (Write-Through)

Consultar o saldo no banco em cada passada de cartão é caro.
- Use um **Redis** para manter o saldo atualizado.
- Quando uma transação ocorre, você atualiza o Redis e o banco de forma síncrona (ou utiliza uma transação atômica) para garantir que o saldo "visto" seja sempre o real.

{: .prompt-warning }
> **Por que não Write-behind?** Embora o modelo de *Write-behind* (escrever apenas no cache e sincronizar com o banco de forma assíncrona depois) ofereça a maior performance, ele é extremamente arriscado para saldos. Se o cache (Redis) falhar antes da persistência no banco, o dinheiro "desaparece" do sistema, gerando uma inconsistência financeira irreparável.

## Estratégia 4: Rate Limiting e Backpressure

Se o sistema começar a degradar, proteja-se.
- Use **Rate Limiting** no API Gateway para evitar que um único cliente sobrecarregue o sistema.
- Use **Backpressure** nos seus consumidores Kafka: se o banco estiver lento, o consumidor deve diminuir o ritmo de leitura automaticamente.

## Insight final

A verdadeira escalabilidade em sistemas de pagamento não é apenas sobre suportar mais requisições por segundo, mas sobre quão bem o seu sistema sobrevive quando as coisas dão errado. Em um cenário de carga extrema, é preferível que um processamento de lote demore 10 minutos a mais do que deixar o banco de dados central indisponível para transações de cartão em tempo real. Priorize sempre a disponibilidade do fluxo crítico sobre a velocidade do fluxo secundário.