---
title: "Isolamento de Transações: READ COMMITTED vs REPEATABLE READ"
date: 2026-01-23 09:00:00 -0300
categories: [Banco de Dados, SQL]
tags: [banco de dados, sql]
render_with_liquid: false
---

Você já teve um bug onde os dados pareciam mudar "sozinhos" no meio do seu código? Isso pode ser falta de compreensão sobre os **Níveis de Isolamento do SQL**. Vamos focar nos dois mais usados: `READ COMMITTED` e `REPEATABLE READ`.

## O Equilíbrio entre Corretude e Performance

Em um mundo perfeito, todas as transações seriam isoladas (Serializable). Mas isso travaria o banco de dados. Por isso, os bancos permitem escolher quão "isolado" você quer estar.

## 1. READ COMMITTED (Padrão do PostgreSQL/Oracle)

Uma transação só vê dados que já foram confirmados (commited) por outras transações.

- **Evita:** Dirty Reads (ler dados "sujos" que podem sofrer rollback).
- **Problema:** **Non-Repeatable Reads**. Se você ler o saldo do usuário no início do método e ler de novo 2 segundos depois (dentro da mesma transação), o valor pode ter mudado porque outra transação fez um commit no meio do caminho.

## 2. REPEATABLE READ (Padrão do MySQL/InnoDB)

Garante que, se você ler um registro, ele será o mesmo até o fim da sua transação, não importa o que aconteça fora dela.

- **Evita:** Non-Repeatable Reads. O banco cria um "snapshot" (versão) dos dados para você.
- **Problema:** **Phantom Reads**. Você pode ver novas linhas inseridas por outras transações que atendem ao seu critério de busca (embora o InnoDB do MySQL mitigue isso com Next-Key Locks).

## Exemplo Visual

1.  **Transação A** começa e lê Saldo = 100.
2.  **Transação B** começa, muda Saldo para 200 e faz **Commit**.
3.  **Transação A** lê o saldo novamente.
    - No `READ COMMITTED`: Verá 200.
    - No `REPEATABLE READ`: Verá 100.

## Qual usar no Spring?

Você pode definir o nível via anotação:
```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void processAudit() { ... }
```

**Dica:** A maioria das aplicações web funciona perfeitamente com `READ COMMITTED`. O `REPEATABLE READ` é mais seguro para processos de auditoria ou relatórios financeiros onde a consistência interna da transação é vital, mas ele consome mais recursos do banco e pode gerar mais `Deadlocks`.

## Conclusão

Isolamento de banco de dados é um trade-off. Quanto maior o isolamento, maior a segurança dos dados, mas menor a concorrência. Conhecer o padrão do seu banco de dados (MySQL vs Postgres) evita bugs silenciosos de concorrência que são impossíveis de reproduzir em ambientes de teste.