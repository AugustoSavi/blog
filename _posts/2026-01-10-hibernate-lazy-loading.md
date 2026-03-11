---
title: "Hibernate Lazy Loading: O que é, como funciona e o perigo da Exception"
date: 2026-01-10 09:00:00 -0300
categories: [Hibernate, JPA]
tags: [hibernate, jpa]
render_with_liquid: false
---

O Hibernate é o rei do ORM no mundo Java, e o **Lazy Loading** (Carregamento Preguiçoso) é uma de suas funcionalidades mais amadas e odiadas. Se você já viu uma `LazyInitializationException` no seu log, este post é para você.

## Performance por Omissão

Imagine que você tem uma classe `User` que tem uma lista de 10.000 `Order`. Se toda vez que você buscasse um usuário o Hibernate buscasse também todos os seus pedidos, sua aplicação ficaria lenta e o banco de dados sobrecarregado. O Lazy Loading resolve isso não buscando os dados até que você realmente precise deles.

## Como funciona o "Truque" do Proxy

O Hibernate não coloca `null` na sua lista quando ela é Lazy. Ele coloca um **Proxy** (um impostor).

```java
@Entity
public class User {
    @OneToMany(fetch = FetchType.LAZY)
    private List<Order> orders;
}

// No código:
User user = repository.findById(1L); // O Hibernate busca o User, mas não as Orders.
// A lista 'orders' aqui é um Proxy (HibernateProxy).
```

Quando você faz `user.getOrders().size()`, o Proxy intercepta a chamada, percebe que os dados ainda não estão lá, e dispara uma nova query no banco de dados para buscar os pedidos. Isso é chamado de **Lazy Initialization**.

## O Terror: LazyInitializationException

O segredo é que o Proxy precisa de uma **Sessão do Hibernate (Session/EntityManager)** ativa para falar com o banco. Se você tentar acessar a lista após a transação ter fechado (ex: dentro do seu Controller ou após o `@Transactional`), você terá um erro:

```text
org.hibernate.LazyInitializationException: could not initialize proxy - no Session
```

## Como Resolver (Do Jeito Certo)

1.  **Join Fetch (JPQL/Criteria):** Informe ao Hibernate que, naquela consulta específica, você quer trazer os filhos de uma vez (Eager).
    ```sql
    SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id
    ```
2.  **Entity Graph:** Uma forma mais elegante e programática de definir o que deve ser carregado.
3.  **DTOs:** Busque apenas os dados necessários e mapeie para um objeto simples (POJO), evitando carregar entidades inteiras.

## Curiosidade: O Problema do N+1

O Lazy Loading é o culpado número 1 pelo problema do **N+1 queries**. Se você buscar 100 usuários e, em um loop, acessar os pedidos de cada um, o Hibernate fará 1 query para os usuários e **100 queries adicionais** para os pedidos. Sempre monitore seus logs de SQL!

## Conclusão

Lazy Loading é uma ferramenta de otimização poderosa, mas exige que você entenda o ciclo de vida da sessão do Hibernate. Use-o com sabedoria e prefira sempre ser explícito sobre o que você quer carregar através de `JOIN FETCH` para evitar surpresas em produção.