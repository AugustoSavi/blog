---
title: "Projetando uma Carteira Digital: Por que você deve usar Double-entry Ledger"
date: 2026-03-02 09:00:00 -0300
categories: [Arquitetura, Fintech]
tags: [wallet, ledger, sql, consistencia, java]
render_with_liquid: false
---

Se você está construindo uma fintech ou qualquer sistema que lide com dinheiro, o erro mais comum (e perigoso) é armazenar o saldo do usuário como uma única coluna `balance` em uma tabela `users`. Por quê? Porque colunas de saldo mentem. O que nunca mente são os **Lançamentos**.

O **Double-entry Ledger** (Livro-razão de Partida Dobrada) é um padrão contábil de 500 anos usado por todos os bancos e fintechs modernas. Ele garante que cada transação financeira seja composta por dois lançamentos: um crédito e um débito, cuja soma deve ser zero.

## A Coluna `balance`

Imagine que você tem a tabela `users` com `balance: 100.00`. 
O usuário faz um pagamento de 50.00. 
Você faz: `UPDATE users SET balance = balance - 50.00 WHERE id = 1;`

Se algo der errado no meio do processo (um timeout, um crash), como você sabe por que o saldo mudou? Como você audita que nenhum centavo desapareceu? Sem um histórico imutável, você está voando no escuro.

## A Solução: Arquitetura de Ledger

Uma arquitetura robusta de carteira digital utiliza três componentes principais:

### 1. Modelo de Dados

```sql
-- Representa as contas (ex: conta corrente, conta benefício)
CREATE TABLE accounts (
    id UUID PRIMARY KEY,
    user_id UUID,
    type VARCHAR(20) -- MEAL, FOOD, MOBILITY
);

-- Representa a transação em si (o contexto)
CREATE TABLE transactions (
    id UUID PRIMARY KEY,
    description TEXT,
    created_at TIMESTAMP
);

-- O coração do sistema: os lançamentos
CREATE TABLE ledger_entries (
    id UUID PRIMARY KEY,
    transaction_id UUID REFERENCES transactions(id),
    account_id UUID REFERENCES accounts(id),
    amount DECIMAL(19, 4), -- Negativo para débito, positivo para crédito
    created_at TIMESTAMP
);
```

### 2. A Regra de Ouro: Soma Zero

Para toda transação, a soma de seus `ledger_entries` deve ser **zero**.

**Exemplo: Pagamento de R$ 50,00 no Restaurante**
- Entrada 1: `-50.00` na `Account_Usuario` (Débito).
- Entrada 2: `+50.00` na `Account_Loja` (Crédito).
- Soma: `-50.00 + 50.00 = 0`.

## Implementação: Consultando o Saldo

O saldo "verdadeiro" é a soma de todos os lançamentos de uma conta. Para performance, você pode ter um cache do saldo (snapshot), mas a fonte da verdade é sempre a soma:

```sql
SELECT SUM(amount) FROM ledger_entries WHERE account_id = 'uuid-da-conta';
```

---

## Funcionamento Interno e Consistência

Para garantir que o saldo nunca fique negativo (Overdraft), você deve usar **Transações ACID** e **Locks** no banco de dados.

```java
@Transactional
public void processTransaction(UUID fromId, UUID toId, BigDecimal amount) {
    // 1. Lock nas contas para evitar Double Spending
    // SELECT * FROM accounts WHERE id IN (fromId, toId) FOR UPDATE;

    // 2. Validar saldo
    BigDecimal currentBalance = ledgerRepository.sumByAccountId(fromId);
    if (currentBalance.compareTo(amount) < 0) {
        throw new InsufficientBalanceException();
    }

    // 3. Criar a Transação
    Transaction tx = transactionRepository.save(new Transaction("Pagamento"));

    // 4. Criar as entradas no Ledger
    ledgerRepository.save(new LedgerEntry(tx.getId(), fromId, amount.negate()));
    ledgerRepository.save(new LedgerEntry(tx.getId(), toId, amount));
}
```

## Curiosidade Técnica: Por que DECIMAL(19, 4)?

Nunca use `Float` ou `Double` para dinheiro. Eles são números de ponto flutuante e sofrem de erros de precisão (ex: `0.1 + 0.2 = 0.30000000000000004`). O tipo `DECIMAL` (ou `NUMERIC`) no SQL e o `BigDecimal` no Java garantem precisão exata, vital para não perder centavos em arredondamentos.

## Conclusão

Projetar uma carteira digital não é sobre guardar um número, é sobre registrar uma história. O Double-entry Ledger traz auditoria, integridade e confiança para o sistema. Se você quer ser um engenheiro de fintech respeitado, esqueça o `balance` e comece a pensar em `ledger_entries`.
