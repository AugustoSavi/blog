---
title: "ACID vs BASE: Do Rigor Bancário à Flexibilidade da Escala"
date: 2026-02-08 09:00:00 -0300
categories: [Banco de Dados, Arquitetura]
tags: [banco de dados, arquitetura]
render_with_liquid: false
---

Quando falamos de bancos de dados, existem dois modelos mentais opostos para garantir a integridade: o tradicional **ACID** (dos bancos relacionais) e o moderno **BASE** (dos bancos NoSQL). Vamos entender qual deles se encaixa melhor no seu microserviço.

## Certeza vs Probabilidade

- No **ACID**, ou o dado está certo e atualizado agora, ou a operação falha.
- No **BASE**, o dado será atualizado em algum momento, e o sistema prioriza estar sempre disponível.

## 1. ACID (Relacional - MySQL, Postgres)
- **Atomicity:** Tudo ou nada.
- **Consistency:** O banco segue todas as regras e constraints.
- **Isolation:** Uma transação não interfere na outra.
- **Durability:** Uma vez salvo, o dado não se perde.
**Ideal para:** Registros financeiros, contabilidade, cadastros críticos.

## 2. BASE (NoSQL - DynamoDB, Cassandra)
- **Basically Available:** O sistema responde quase sempre.
- **Soft state:** O estado pode mudar sem uma entrada direta (devido à propagação de dados).
- **Eventual consistency:** O dado ficará consistente "eventualmente" (em milissegundos ou segundos).
**Ideal para:** Logs, redes sociais, catálogos de produtos, sistemas de alta escala.

## Exemplo Visual: Like no Instagram

Quando você dá um "like" em um post, o seu amigo pode não ver o contador aumentar instantaneamente (Eventual Consistency). Isso é o modelo **BASE**. O sistema não vai travar o mundo só para garantir que 1 bilhão de pessoas vejam o mesmo número de likes no exato milissegundo.

## Por que usar os dois?

- Para o **Saldo do Usuário**, use **ACID** (MySQL). Não podemos ter "eventualidade" quando o assunto é dinheiro na conta.
- Para o **Histórico de Notificações** ou logs de auditoria rápida, o **BASE** (DynamoDB) escala muito melhor e atende perfeitamente à necessidade.

## Conclusão

Não escolha tecnologias por "hype". Escolhem pelo modelo de consistência que o negócio exige. Entender o trade-off entre o rigor do ACID e a escala do BASE é fundamental para desenhar sistemas que não quebram sob carga.