---
title: "Kotlin Coroutines vs Threads: Por que menos é mais na concorrência?"
date: 2026-02-03 09:00:00 -0300
categories: [Kotlin, Coroutines]
tags: [kotlin, coroutines]
render_with_liquid: false
---

Se você precisa rodar 10.000 tarefas simultâneas no Java tradicional, você provavelmente terá problemas de memória (cada Thread ocupa cerca de 1MB de stack). No Kotlin, você pode rodar 100.000 **Coroutines** no mesmo hardware sem nem suar. Qual o segredo?

## Suspensão vs Bloqueio

A diferença fundamental está no que acontece quando o código precisa esperar (por exemplo, uma resposta de API).
- **Thread:** Fica "bloqueada". O sistema operacional gasta recursos mantendo aquela thread parada, sem fazer nada, esperando o I/O.
- **Coroutine:** Fica "suspensa". Ela libera a thread em que estava rodando para que outras tarefas possam usá-la. Quando o I/O termina, a coroutine "acorda" e volta a rodar de onde parou.

## Exemplo Visual em Código

```kotlin
// Usando Threads (Bloqueante)
fun fetchUser() {
    val user = api.getUser() // Thread fica presa aqui
    println(user)
}

// Usando Coroutines (Não Bloqueante)
suspend fun fetchUser() {
    val user = api.getUserSuspended() // Coroutine suspende, Thread fica livre!
    println(user)
}
```

*Explicação:* A palavra chave `suspend` avisa o compilador que aquela função pode pausar a execução sem travar a thread subjacente.

## Structured Concurrency

O Kotlin introduz o conceito de **Concorrência Estruturada**. Isso significa que, se você disparar 10 coroutines dentro de um escopo e o escopo for cancelado (ex: o usuário fechou a tela ou a requisição HTTP expirou), todas as 10 coroutines são canceladas automaticamente. Isso evita vazamentos de memória (memory leaks) e tarefas "zumbis" rodando em background.

## Curiosidade: Threads de 1MB vs Coroutines de Bytes

Uma Thread é um recurso caro do Sistema Operacional. Uma Coroutine é apenas um objeto gerenciado pela JVM. O custo de trocar de contexto entre Threads (Context Switch) é alto. O custo de suspender uma Coroutine é quase zero.

## Conclusão

Coroutines são a evolução natural da concorrência na JVM. Elas permitem escrever código assíncrono que parece síncrono, mantendo a legibilidade e a alta performance necessária para microserviços modernos.