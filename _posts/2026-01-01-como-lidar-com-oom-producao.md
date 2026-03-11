---
title: "OutOfMemoryError"
date: 2026-01-01 09:00:00 -0300
categories: [Java, JVM]
tags: [java, jvm]
render_with_liquid: false
---

Você está em um plantão tranquilo quando, de repente, o alerta do monitoramento dispara: `java.lang.OutOfMemoryError: Java heap space`. O serviço parou de responder, as transações estão falhando e a pressão aumenta. O que você faz? Reinicia o pod e reza para não acontecer de novo ou investiga a fundo?

## Quando a Memória Transborda

O `OutOfMemoryError` (OOM) ocorre quando a Java Virtual Machine (JVM) não consegue mais alocar memória para novos objetos e o Garbage Collector não consegue liberar espaço suficiente. Em produção, isso geralmente é sintoma de um **Memory Leak** (vazamento de memória) ou de um dimensionamento incorreto dos recursos (Heap muito pequeno para a carga atual).

## Ferramentas de Combate

Para resolver um OOM, você precisa de dados. Aqui estão as ferramentas essenciais:

1.  **Heap Dump:** Um "instantâneo" da memória no momento da falha.
2.  **VisualVM / JVisualVM:** Para análise em tempo real (em ambientes de dev/staging).
3.  **Eclipse MAT (Memory Analyzer Tool):** A ferramenta para analisar Heap Dumps e encontrar "vazadores" de memória.
4.  **JProfiler / YourKit:** Ferramentas comerciais poderosas para análise profunda.
5.  **Datadog / New Relic / Prometheus:** Para monitorar tendências de consumo de memória antes da explosão.

## Estratégia de Investigação

A regra número 1 é: **Não reinicie sem antes coletar o Heap Dump.**

```bash
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/logs/dumps/oom_dump.hprof
```
*Explicação:* Essa configuração garante que, ao morrer por falta de memória, a JVM salve o estado atual em um arquivo `.hprof`. Sem isso, você estará "caçando fantasmas".

## Analisando o Heap Dump com Eclipse MAT

Ao abrir o dump no MAT, procure pelo **Leak Suspects Report**. Ele mostrará quais objetos estão ocupando a maior parte da memória.

Muitas vezes, o culpado é:
- Um cache estático que nunca é limpo.
- Uma lista de conexões que não foram fechadas.
- Objetos grandes sendo carregados do banco de dados sem paginação.

## Curiosidades Técnicas: Nem todo OOM é igual

Existem vários tipos de `OutOfMemoryError`:
- **Java heap space:** Objetos demais no Heap.
- **GC Overhead limit exceeded:** O GC está gastando 98% do tempo para liberar menos de 2% da memória (está "patinando").
- **Metaspace:** Memória nativa usada para metadados de classes (comum em deploys excessivos sem reiniciar a JVM).

## Checklist de Sobrevivência ao OOM

Antes de considerar o incidente encerrado, certifique-se de que sua infraestrutura segue estes pontos:
- [ ] `-XX:+HeapDumpOnOutOfMemoryError` está ativo em todos os ambientes críticos.
- [ ] O volume de logs/dumps possui espaço em disco suficiente para o tamanho do Heap.
- [ ] Existe monitoramento de métricas de JVM (Heap, Metaspace, GC) com alertas de tendência.
- [ ] A equipe sabe operar o Eclipse MAT ou similar para análise post-mortem.
- [ ] O limite de memória do container (K8s/Docker) é superior ao `-Xmx` para evitar OOM Kill do SO.