---
title: "Ciclo de Vida das Entidades JPA: Entenda os 4 Estados"
date: 2026-01-17 09:00:00 -0300
categories: [Hibernate, JPA]
tags: [hibernate, jpa]
render_with_liquid: false
---

No JPA/Hibernate, uma entidade não é apenas um objeto Java comum. Ela passa por diferentes estados dentro do **Persistence Context**. Entender esses estados é a chave para evitar bugs bizarros onde dados não são salvos ou exceções inesperadas acontecem.

## Onde está o seu objeto?

Imagine o Persistence Context (EntityManager) como um "vigia". Ele decide quem ele está monitorando e quem ele ignora.

## Os 4 Estados Fundamentais

1.  **Transient (Transitório):**
    O objeto foi acabado de ser criado com `new`. Ele ainda não tem ID e o Hibernate não sabe que ele existe. Se você mudar um valor aqui, nada acontece no banco.
    ```java
    User user = new User(); // Transient
    ```

2.  **Persistent / Managed (Gerenciado):**
    O objeto está associado a uma sessão ativa e tem um ID no banco. Qualquer alteração feita nele será detectada pelo **Dirty Checking** e salva automaticamente no banco no final da transação.
    ```java
    User user = em.find(User.class, 1L); // Managed
    user.setName("Novo Nome"); // Será salvo no commit!
    ```

3.  **Detached (Desconectado):**
    O objeto já foi gerenciado, mas a sessão foi fechada ou ele foi removido explicitamente (`em.detach()`). O objeto ainda tem um ID, mas o Hibernate não o vigia mais. Alterações aqui **não** vão para o banco automaticamente.
    ```java
    em.close(); // user agora é Detached
    ```

4.  **Removed (Removido):**
    O objeto está marcado para ser deletado do banco de dados no final da transação.
    ```java
    em.remove(user); // Removed
    ```

## O Truque do merge()

Se você tem um objeto `Detached` e quer que ele volte a ser `Managed`, você usa o `merge()`. Mas atenção: o `merge` não altera o seu objeto; ele cria uma **nova cópia** gerenciada com os dados que você passou.

## Por que isso importa?

Muitos erros de "entidade não salva" acontecem porque o desenvolvedor está alterando um objeto que já está no estado `Detached`. Outro erro comum é carregar milhares de objetos como `Managed` desnecessariamente, consumindo muita memória do "vigia" (Persistence Context).

## Checklist de Estados da Entidade

Sempre que uma entidade não se comportar como esperado, valide estes pontos:
- [ ] O objeto foi criado com `new`? (Estado: **Transient** - chame `persist` ou `save`).
- [ ] O método está dentro de um `@Transactional` ativo? (Estado: **Managed** - o Dirty Checking funcionará).
- [ ] A sessão foi fechada ou o objeto veio de um cache externo? (Estado: **Detached** - use `merge`).
- [ ] Você chamou `repository.delete()`? (Estado: **Removed** - o objeto ainda existe na memória até o flush).
- [ ] O ID da entidade já está preenchido antes de salvar? (Pode causar um `merge` inesperado em vez de `persist`).