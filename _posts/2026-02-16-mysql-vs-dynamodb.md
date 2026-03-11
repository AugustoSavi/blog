---
title: "MySQL vs DynamoDB: Onde salvar os dados financeiros?"
date: 2026-02-16 09:00:00 -0300
categories: [Arquitetura de Dados, MySQL]
tags: [arquitetura de dados, mysql, dynamodb]
render_with_liquid: false
---

Em sistemas complexos, lidamos com diferentes tipos de dados. Alguns exigem o rigor de uma transação bancária, outros precisam da velocidade da luz em escala global. Como escolher entre o relacional MySQL e o NoSQL DynamoDB?

## Nem todo dado é igual

Se você errar o saldo de um benefício por 1 centavo, você tem um problema jurídico. Se você atrasar a atualização do nome de perfil do usuário por 1 segundo, ninguém percebe.

## 1. Quando usar MySQL (Relacional / ACID)

O MySQL é o lugar para o **Core Financeiro**.
- **Cenário:** Transferência de saldo entre categorias (ex: de Refeição para Alimentação).
- **Por que:** Você precisa de uma **Transação Atômica**. Ou o dinheiro sai de um lugar e entra no outro, ou nada acontece. O MySQL garante que o saldo nunca fique "no limbo".
- **Complexidade:** Se precisar de JOINs complexos para relatórios de conformidade (compliance), o SQL é imbatível.

## 2. Quando usar DynamoDB (NoSQL / Chave-Valor)

O DynamoDB é o lugar para **Eventos e Alta Escala**.
- **Cenário:** Log de cada tentativa de uso do cartão (autorizações, negadas, erros de senha).
- **Por que:** Milhões de transações geram um volume de logs imenso. O DynamoDB escala horizontalmente de forma quase infinita com latência constante de milissegundos.
- **Trade-off:** A consistência é **eventual** por padrão (embora aceite leitura consistente, é mais caro). Ideal para dados que você escreve muito e lê por uma chave específica (`transactionId`).

## O Modelo Híbrido

A resposta correta muitas vezes é **ambos**:
1. O cartão passa. O sistema valida o saldo no **Redis/MySQL** (Consistência Forte).
2. A transação é aprovada. Um evento é disparado para o Kafka.
3. Um consumidor salva o detalhe rico da transação no **DynamoDB** para histórico de extrato (Alta Escala).

## Conclusão: O cofre e o armazém

Pense no **MySQL** como o cofre de um banco: ele é rígido, altamente seguro, garante que nada saia sem registro e que cada centavo esteja no lugar certo. Já o **DynamoDB** é como um imenso armazém logístico: ele foca em velocidade de entrada e saída, escala horizontalmente conforme o estoque cresce e permite que você encontre qualquer item instantaneamente, desde que saiba o código da prateleira. Use o cofre para o patrimônio e o armazém para a operação em massa.