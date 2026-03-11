---
title: "O Problema do N+1: Como uma query inofensiva destrói sua performance"
date: 2026-02-05 09:00:00 -0300
categories: [Hibernate, JPA]
tags: [hibernate, jpa]
render_with_liquid: false
---

Você tem 100 usuários e quer listar o nome de cada um com o título do seu último pedido. Você faz a query dos usuários e, em um loop, busca o pedido de cada um. Parabéns, você acabou de criar o **Problema do N+1**, o pesadelo número 1 de performance em aplicações ORM.

## A Armadilha do Loop

O código parece limpo:
```java
List<User> users = repository.findAll(); // 1 query
for (User user : users) {
    System.out.println(user.getOrders().get(0).getTitle()); // N queries!
}
```

Se você tem 100 usuários, o Hibernate fará:
- 1 query para buscar os usuários.
- **100 queries** (uma para cada usuário) para buscar os pedidos.
Total: 101 queries para um dado que poderia ter sido resolvido em uma única operação.

## Como resolver: JOIN FETCH

A solução é dizer ao Hibernate: "Quando buscar os usuários, já traga os pedidos em uma única query usando um JOIN".

```sql
-- No seu Repository JPA
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();
```
*Explicação:* O `FETCH` instrui o JPA a carregar a coleção associada imediatamente em uma única query SQL com JOIN, evitando as idas e vindas ao banco dentro do loop.

## Entity Graph: A Solução Elegante

Se você não quer escrever JPQL, pode usar um `EntityGraph`:
```java
@EntityGraph(attributePaths = {"orders"})
List<User> findAll();
```
Isso faz a mesma coisa: força o carregamento imediato (Eager) apenas para essa chamada específica, mantendo o `Lazy Loading` como padrão para o resto do sistema.

## Curiosidade: Por que o Hibernate não faz isso sozinho?

Porque se ele fizesse JOIN em tudo por padrão, você acabaria trazendo o banco de dados inteiro para a memória em uma consulta simples. O `Lazy Loading` é uma proteção, mas exige que o desenvolvedor saiba quando "quebrar" essa proteção para otimizar a performance.

## Conclusão

Sempre monitore o SQL gerado pela sua aplicação em desenvolvimento. Se você vir o mesmo `SELECT` se repetindo centenas de vezes com IDs diferentes, você encontrou o N+1. Use `JOIN FETCH` ou `EntityGraph` e seu banco de dados (e o seu usuário) vão te agradecer.