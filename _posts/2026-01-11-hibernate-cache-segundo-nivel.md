---
title: "Hibernate Cache de 2º Nível: Escalando sua Persistência"
date: 2026-01-11 09:00:00 -0300
categories: [Hibernate, JPA]
tags: [hibernate, jpa]
render_with_liquid: false
---

Se o banco de dados é o gargalo da sua aplicação, você provavelmente já pensou em usar cache. O Hibernate oferece um mecanismo poderoso chamado **Cache de Segundo Nível (L2 Cache)**. Mas cuidado: ele não é uma bala de prata e pode trazer complexidades inesperadas.

## Além da Sessão

Diferente do Cache de 1º Nível (que vive apenas enquanto a `Session` está aberta), o Cache de 2º Nível é compartilhado por toda a aplicação (ou até entre múltiplos nós de um cluster). Isso significa que se um usuário buscar o `Produto A`, o segundo usuário que buscar o mesmo produto poderá recebê-lo direto da memória, sem tocar no banco.

## Como funciona a Hierarquia

1.  **Cache de 1º Nível (L1):** Obrigatório. Escopo de transação.
2.  **Cache de 2º Nível (L2):** Opcional. Escopo de `SessionFactory` (aplicação).
3.  **Cache de Query:** Opcional. Armazena o resultado de consultas específicas (IDs resultantes).

## Implementações Comuns

O Hibernate não implementa o cache sozinho; ele fornece as interfaces para você plugar provedores como:
- **Ehcache:** Muito comum para aplicações standalone.
- **Infinispan:** Excelente para ambientes distribuídos/clusterizados.
- **Hazelcast:** Outra opção robusta para sistemas distribuídos.

## Exemplo de Configuração

Para habilitar, você precisa de três coisas:
1. Adicionar o provedor no `pom.xml`.
2. Habilitar nas propriedades do Hibernate:
```properties
hibernate.cache.use_second_level_cache=true
hibernate.cache.region.factory_class=org.hibernate.cache.jcache.internal.JCacheRegionFactory
```
3. Marcar suas entidades:
```java
@Entity
@Cacheable
@org.hibernate.annotations.Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product { ... }
```

## Estratégias de Concorrência

- **READ_ONLY:** Para dados que nunca mudam. Mais rápido.
- **NONSTRICT_READ_WRITE:** Se a consistência absoluta não for crítica.
- **READ_WRITE:** Garante consistência (usa locks). Ideal para a maioria dos casos.
- **TRANSACTIONAL:** Para transações JTA complexas.

## Dados Obsoletos (Stale Data)

O maior desafio do L2 Cache é a invalidação. Se você atualizar um dado diretamente no banco (via SQL puro), o Hibernate não saberá e continuará servindo a versão antiga que está no cache. Portanto, use L2 Cache apenas quando a maior parte das alterações nos dados passar pelo próprio Hibernate.

## Conclusão

O Cache de 2º Nível pode reduzir drasticamente a carga no banco de dados, mas exige um design cuidadoso, especialmente em relação à concorrência e invalidação. Comece otimizando suas queries e use o cache como o próximo passo para escala.