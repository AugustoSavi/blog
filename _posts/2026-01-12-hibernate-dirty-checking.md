---
title: "Hibernate Dirty Checking"
date: 2026-01-12 09:00:00 -0300
categories: [Hibernate, JPA]
tags: [hibernate, jpa]
render_with_liquid: false
---

Você já reparou que, às vezes, você altera um objeto Java recuperado do Hibernate, chama o `commit()` da transação e, magicamente, o banco de dados é atualizado sem você ter chamado `repository.save()` ou `update()`? Esse "truque" se chama **Dirty Checking**.

## O Hibernate está te Observando

O Dirty Checking (Verificação de Sujeira) é o mecanismo pelo qual o Hibernate detecta se uma entidade carregada na memória foi modificada e precisa ser sincronizada com o banco de dados.

## Como funciona internamente?

Quando o Hibernate carrega uma entidade do banco de dados para a `Session` (Cache de 1º Nível), ele guarda duas coisas:
1.  O objeto que ele te entrega.
2.  Uma **cópia idêntica** (snapshot) do estado original daquele objeto.

No momento do `flush` (geralmente antes do commit), o Hibernate percorre todos os objetos na sessão e compara cada um com seu snapshot original. Se houver qualquer diferença, ele marca o objeto como "sujo" (dirty) e gera automaticamente o comando `UPDATE` necessário.

## Exemplo Prático

```java
@Transactional
public void updateStatus(Long userId) {
    User user = repository.findById(userId).get(); // 1. Carrega e tira snapshot
    user.setStatus("ACTIVE"); // 2. Altera o objeto na memória
    // Não chamei save()!
} // 3. Ao sair do @Transactional, o Hibernate faz o Dirty Checking e executa o UPDATE.
```

## O Custo da Magia

Embora muito prático, o Dirty Checking tem um custo de performance. Se você carregar 10.000 objetos em uma única transação, o Hibernate terá que comparar 10.000 snapshots no final.

**Dicas de Otimização:**
- **ReadOnly Transactions:** Se você marcar o método como `@Transactional(readOnly = true)`, o Hibernate pode desabilitar o Dirty Checking, economizando tempo e memória.
- **StatelessSession:** Para processamentos em lote (batch), onde você não precisa desse monitoramento automático.

## Curiosidade: Dynamic Update

Por padrão, o `UPDATE` gerado pelo Hibernate inclui **todas as colunas** da tabela, mesmo as que não mudaram. Se você quiser que ele envie apenas as colunas alteradas, use a anotação `@DynamicUpdate` na sua entidade. Isso pode ajudar em tabelas com muitas colunas ou alto volume de escritas.

## O Princípio da Sincronização Automática

No Hibernate, a unidade de trabalho (`Session`) é a dona da verdade sobre o estado das suas entidades. A regra é clara: qualquer alteração em um objeto **monitorado** será persistida no banco de dados ao final da transação, independentemente de chamadas explícitas a métodos de salvamento. Para evitar efeitos colaterais indesejados, sempre utilize transações de "somente leitura" em fluxos de consulta e evite manter objetos de entidade abertos além do necessário na memória.