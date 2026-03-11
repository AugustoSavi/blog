---
title: "Blue-Green Deployment: Como atualizar seu sistema sem ninguém perceber"
date: 2026-01-26 09:00:00 -0300
categories: [DevOps, CI]
tags: [devops, ci, cd]
render_with_liquid: false
---

O medo de fazer o deploy de uma nova versão e derrubar o sistema é real. O **Blue-Green Deployment** é uma técnica de liberação (release) que reduz esse risco a quase zero, permitindo que você volte atrás instantaneamente se algo der errado.

## O Jogo dos Dois Ambientes

Imagine que você tem dois ambientes de produção idênticos. Chamamos um de **Blue** (o que está no ar agora) e o outro de **Green** (o que vai receber a versão nova).

## Como funciona o fluxo

1.  **Estado Atual:** Todo o tráfego dos usuários está indo para o ambiente **Blue** (v1.0).
2.  **Preparação:** Você faz o deploy da nova versão (v1.1) no ambiente **Green**. Este ambiente está isolado, sem receber tráfego público.
3.  **Teste em Produção:** Sua equipe de QA e desenvolvedores acessam o ambiente **Green** (via uma URL interna ou VPN) e verificam se tudo está funcionando no hardware real.
4.  **A Virada (Switch):** Quando todos estão confiantes, o Roteador/Load Balancer muda o tráfego de **Blue** para **Green** instantaneamente.
5.  **Monitoramento:** Se o ambiente **Green** apresentar erros inesperados, basta mudar o roteador de volta para **Blue**. O rollback é imediato.

## O Grande Desafio: O Banco de Dados

O Blue-Green é fácil para código (stateless), mas difícil para o banco de dados. Se a v1.1 mudar o esquema do banco (deletar uma coluna, por exemplo), a v1.0 (Blue) vai quebrar se você precisar fazer o rollback.

**A Solução:** Mudanças no banco devem ser sempre **retrocompatíveis**. Siga o padrão *Expand and Contract*:
1.  **Expand:** Adicione a nova coluna (ambas as versões funcionam).
2.  **Migrate:** Mova os dados.
3.  **Contract:** Delete a coluna antiga (só depois que a v1.0 não existir mais).

## Blue-Green vs Canary

Enquanto o Blue-Green vira 100% do tráfego de uma vez, o **Canary Deployment** libera a nova versão para apenas 5% ou 10% dos usuários inicialmente, aumentando gradualmente conforme a confiança aumenta.

## Conclusão

Blue-Green Deployment traz paz de espírito para a equipe de engenharia e disponibilidade contínua para o usuário. Ele exige maturidade de infraestrutura (automação) e cuidado redobrado com migrações de banco de dados, mas o benefício de um "zero-downtime deploy" vale cada centavo do investimento.