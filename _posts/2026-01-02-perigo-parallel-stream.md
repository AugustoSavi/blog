---
title: "O Perigo Escondido do parallelStream(): Quando o Rápido se Torna Lento"
date: 2026-01-02 09:00:00 -0300
categories: [Java]
tags: [java]
render_with_liquid: false
---

Você tem uma lista enorme de Objetos e precisa processá-las. Alguém sugere: "Basta usar `.parallelStream()` e vai voar!". Parece mágica, certo? Mas em ambientes de servidores de aplicação, essa "mágica" pode derrubar sua performance e até travar seu sistema.

## Onde está o gargalo?

O Java 8 introduziu os Streams Paralelos para facilitar o processamento em múltiplos núcleos. No entanto, o que muitos esquecem é que, por baixo dos panos, todos os `parallelStream()` da sua aplicação compartilham o **mesmo pool de threads**: o `ForkJoinPool.commonPool()`.

## Pool Compartilhado

Imagine um servidor rodando Spring Boot com 100 usuários simultâneos. Se cada requisição disparar um `parallelStream()`, todas estarão lutando pelas mesmas threads do `commonPool`. Se uma dessas threads travar por I/O bloqueante (como uma chamada de API lenta ou query de banco), ela retira essa thread do pool global, afetando partes da aplicação que nem usam o stream paralelo.

## Exemplo Prático: O Cenário de Desastre

```java
// Código aparentemente inofensivo
public List<Result> processTransactions(List<Transaction> transactions) {
    return transactions.parallelStream()
        .map(tx -> callExternalApi(tx)) // I/O Bloqueante dentro do Parallel Stream!
        .collect(Collectors.toList());
}
```

*Explicação:* O `callExternalApi` é uma operação de rede. Se a API externa demorar, as threads do `ForkJoinPool.commonPool()` ficarão presas esperando. Como o pool é limitado pelo número de processadores (cores - 1), você rapidamente esgota a capacidade da JVM de processar qualquer outra tarefa paralela.

## Quando usar parallelStream()?

1.  **Operações CPU-Intensive:** Cálculos matemáticos complexos sem I/O.
2.  **Datasets Gigantes:** Onde o custo de gerenciar threads é menor que o ganho de performance.
3.  **Ambientes Isolados:** Onde você sabe que ninguém mais está usando o `commonPool`.

## Boas Práticas e Alternativas

Se você REALMENTE precisa de paralelismo para I/O, não use o pool comum. Crie seu próprio `ExecutorService` ou use bibliotecas de programação reativa (Project Reactor, RxJava) ou, se estiver em Java moderno, explore os **Virtual Threads** (Project Loom).

```java
// Alternativa com Virtual Threads (Java 21+)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<CompletableFuture<Result>> futures = transactions.stream()
        .map(tx -> CompletableFuture.supplyAsync(() -> callExternalApi(tx), executor))
        .toList();
    // ... aguarda resultados
}
```

## Paralelismo em Streams

Nunca utilize `parallelStream()` para operações que envolvam I/O bloqueante ou recursos compartilhados globalmente. Reserve o paralelismo implícito exclusivamente para tarefas **CPU-bound** puras e isoladas. Se a thread precisa esperar por algo externo, você deve assumir o controle total do ciclo de vida dessas threads através de executores dedicados ou modelos reativos.