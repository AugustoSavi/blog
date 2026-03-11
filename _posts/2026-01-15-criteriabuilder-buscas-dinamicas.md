---
title: "Buscas Dinâmicas com CriteriaBuilder"
date: 2026-01-15 09:00:00 -0300
categories: [Hibernate, JPA]
tags: [hibernate, jpa]
render_with_liquid: false
---

Você precisa criar uma tela de busca com 10 filtros opcionais: nome, data de início, data de fim, status, categoria... Se você tentar resolver isso com strings (JPQL) e um monte de `if (filtro != null)`, seu código vai se tornar um monstro impossível de manter. O **CriteriaBuilder** do JPA é a solução elegante para esse problema.

## Queries Tipo Lego

A API de Criteria permite construir consultas de forma programática e "type-safe" (segura em relação aos tipos). É como montar um conjunto de Lego: você só adiciona as peças (cláusulas WHERE) que o usuário realmente selecionou.

## Exemplo Prático: Busca de Transações

Imagine que queremos buscar transações filtrando opcionalmente por `userId`, `minAmount` e `status`.

```java
public List<Transaction> findTransactions(String userId, Double minAmount, Status status) {
    CriteriaBuilder cb = entityManager.getCriteriaBuilder();
    CriteriaQuery<Transaction> cq = cb.createQuery(Transaction.class);
    Root<Transaction> root = cq.from(Transaction.class);
    
    List<Predicate> predicates = new ArrayList<>();

    if (userId != null) {
        predicates.add(cb.equal(root.get("userId"), userId));
    }
    if (minAmount != null) {
        predicates.add(cb.greaterThanOrEqualTo(root.get("amount"), minAmount));
    }
    if (status != null) {
        predicates.add(cb.equal(root.get("status"), status));
    }

    cq.where(predicates.toArray(new Predicate[0]));
    
    return entityManager.createQuery(cq).getResultList();
}
```

## Por que isso é melhor que String?

1.  **Sem Erros de Sintaxe:** Como você usa métodos em vez de Strings, o compilador te ajuda. Você não vai esquecer um espaço antes do `AND`.
2.  **Refatoração Amigável:** Se você mudar o nome de um campo na entidade, ferramentas como o Hibernate Metamodel podem gerar classes que permitem fazer `root.get(Transaction_.AMOUNT)`, garantindo que o erro apareça no tempo de compilação.
3.  **Modularidade:** Você pode criar métodos que retornam `Predicate` e reutilizá-los em diferentes partes da aplicação.

## Trade-off: Verbosidade

O maior defeito da Criteria API é ser muito verbosa. Para uma query simples e estática, um `@Query` no Spring Data é muito mais rápido de escrever. Use Criteria apenas quando a query for **realmente dinâmica**.

## Dica: Specification do Spring Data

Se você usa Spring Data JPA, você pode usar a interface `Specification`. Ela abstrai a parte chata do `CriteriaBuilder` e permite que você escreva filtros reutilizáveis de forma muito mais limpa:

```java
repository.findAll(specUserId(id).and(specMinAmount(100)));
```

## Conclusão: Em Resumo

O `CriteriaBuilder` transforma o caos de strings concatenadas em uma estrutura de consulta modular e segura, garantindo que buscas complexas permaneçam manuteníveis e protegidas contra erros de sintaxe em tempo de execução.