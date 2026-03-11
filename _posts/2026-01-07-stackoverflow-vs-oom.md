---
title: "StackOverflow vs OutOfMemory: Qual a Diferença?"
date: 2026-01-07 09:00:00 -0300
categories: [Java, JVM]
tags: [java, jvm]
render_with_liquid: false
---

Sua aplicação caiu. No log, um erro que começa com `java.lang.`. Mas foi um `StackOverflowError` ou um `OutOfMemoryError`? Embora ambos sejam problemas de memória, eles indicam falhas em lugares e por motivos completamente diferentes.

## Onde a Memória Acabou?

Pense na memória da JVM como um escritório. A **Stack** são as mesas individuais (pequenas e organizadas), e o **Heap** é o depósito central (grande e bagunçado).

## 1. StackOverflowError: O Fim da Pilha

Este erro ocorre na **Stack** de uma thread específica.

- **Causa Comum:** Recursão infinita ou muito profunda.
- **O que acontece:** Cada vez que um método é chamado, ele ocupa um espaço (frame) na pilha. Se você chama métodos sem parar, a pilha atinge o seu limite físico.
- **Exemplo Clássico:**
```java
public void recursiveMethod() {
    recursiveMethod(); // Nunca para, empilha frames até estourar.
}
```
*Explicação:* O erro acontece porque a thread tentou usar mais espaço do que o configurado em `-Xss` (tamanho da stack por thread).

## 2. OutOfMemoryError: O Depósito Lotado

Este erro geralmente ocorre no **Heap**, que é compartilhado por todas as threads.

- **Causa Comum:** Criar objetos demais ou manter referências a objetos que não são mais usados (Memory Leak).
- **O que acontece:** O Garbage Collector tenta limpar o Heap, mas não encontra espaço suficiente para alocar um novo objeto solicitado pela aplicação.
- **Exemplo Clássico:**
```java
List<byte[]> list = new ArrayList<>();
while(true) {
    list.add(new byte[1024 * 1024]); // Adiciona 1MB por vez até lotar o Heap.
}
```
*Explicação:* O erro acontece porque a aplicação tentou usar mais espaço do que o configurado em `-Xmx` (tamanho máximo do heap).

## Comparação Rápida

| Característica | StackOverflowError | OutOfMemoryError |
| :--- | :--- | :--- |
| **Local** | Stack (Pilha da Thread) | Heap (Depósito de Objetos) |
| **Escopo** | Afeta apenas uma thread | Pode afetar a aplicação toda |
| **Configuração** | `-Xss` | `-Xmx` / `-Xms` |
| **Diagnóstico** | Stack Trace (fácil de ver o loop) | Heap Dump (precisa de análise) |

## Curiosidade Técnica

Existem tipos especiais de `OutOfMemoryError` que não são no Heap, como o `Metaspace` (onde ficam as definições das classes). Já o `StackOverflowError` é quase sempre relacionado à lógica de chamadas de métodos.

## Conclusão

Se o log mostra `StackOverflow`, verifique seus algoritmos recursivos. Se mostra `OutOfMemory`, verifique vazamentos de memória e o tamanho dos seus caches. Entender o local do erro é o primeiro passo para uma correção rápida.