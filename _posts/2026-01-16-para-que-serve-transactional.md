---
title: "Para que serve o @Transactional? Muito além do Begin e Commit"
date: 2026-01-16 09:00:00 -0300
categories: [Spring Framework]
tags: [spring framework]
render_with_liquid: false
---

Se você trabalha com Spring, o `@Transactional` é provavelmente a anotação que você mais usa. Mas você sabe o que acontece "por baixo dos panos" quando você a coloca em um método? Ela faz muito mais do que apenas abrir e fechar uma transação no banco.

## A Magia do Proxy

Quando você anota um método com `@Transactional`, o Spring não executa o seu código diretamente. Ele cria um **Proxy** (um intermediário). Quando alguém chama o seu método, o Proxy intercepta a chamada e:
1.  Inicia uma transação no banco de dados.
2.  Executa o seu código.
3.  Se tudo der certo, faz o `commit`.
4.  Se ocorrer uma `RuntimeException`, faz o `rollback`.

## O que ela garante? (ACID)

O `@Transactional` garante que sua operação seja **Atômica**: ou tudo o que está dentro do método acontece, ou nada acontece. Isso é vital para transferências financeiras, por exemplo.

## Propriedades Importantes

Muitos desenvolvedores ignoram as configurações extras:

- **Propagation:** Define o que acontece se um método transacional chama outro. O padrão é `REQUIRED` (usa a transação existente ou cria uma nova). Existe também o `REQUIRES_NEW` (sempre cria uma nova transação, suspendendo a atual).
- **Isolation:** Define o nível de isolamento entre transações simultâneas (ex: `READ_COMMITTED`).
- **ReadOnly:** Uma dica para o Hibernate otimizar a performance, desabilitando o Dirty Checking.
- **RollbackFor:** Por padrão, o Spring só faz rollback para `RuntimeException`. Se você quiser que ele faça para exceções checadas (`Exception`), precisa configurar aqui.

## A Armadilha da Chamada Interna (Self-Invocation)

Este é o erro mais comum:
```java
@Service
public class MyService {
    public void methodA() {
        methodB(); // O @Transactional de methodB SERÁ IGNORADO!
    }

    @Transactional
    public void methodB() { ... }
}
```
*Explicação:* Como o Spring usa Proxies, o `@Transactional` só funciona quando a chamada vem de **fora** da classe. Uma chamada interna ignora o Proxy e, consequentemente, a transação.

## Conclusão

O `@Transactional` é uma ferramenta poderosa de abstração, mas exige cuidado. Entender como os Proxies funcionam e como configurar a propagação corretamente é o que garante sistemas confiáveis.