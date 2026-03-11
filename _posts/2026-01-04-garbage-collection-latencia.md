---
title: "Entendendo o Garbage Collection e as Pausas Stop-the-World"
date: 2026-01-04 09:00:00 -0300
categories: [Java, JVM]
tags: [java, jvm]
render_with_liquid: false
---

Muitos desenvolvedores veem o Garbage Collector (GC) como um zelador silencioso que limpa a memória para nós. Mas em sistemas de alta performance, esse "zelador" pode se tornar o seu maior inimigo se ele decidir parar tudo o que você está fazendo para limpar o chão.

## A Pausa Indesejada

Você já percebeu picos de latência aleatórios na sua aplicação? Aqueles milissegundos (ou segundos) onde tudo parece congelar? Isso muitas vezes é causado pelas pausas **Stop-the-World (STW)** do Garbage Collector.

## O Que é Stop-the-World?

Para que o GC possa mover objetos de lugar ou identificar o que pode ser deletado sem riscos, ele precisa garantir que ninguém esteja alterando a memória naquele instante. Por isso, a JVM suspende todas as threads da aplicação. Em sistemas financeiros ou de baixa latência, isso é crítico.

## Como o GC se Organiza (Generational Hypothesis)

A maioria dos objetos em Java "morre jovem". Por isso, o GC divide o Heap em:
1.  **Young Generation (Eden + Survivor):** Onde os objetos nascem. Coletas aqui são frequentes e geralmente rápidas (Minor GC).
2.  **Old Generation:** Para onde vão os sobreviventes. Coletas aqui são mais pesadas e demoradas (Major GC / Full GC).

## Tipos de GC e Trade-offs

Não existe o "melhor" GC, existe o melhor para o seu caso de uso:

- **G1 (Garbage First):** O padrão moderno. Tenta equilibrar throughput e latência. Ideal para heaps grandes (acima de 4GB) onde você quer pausas previsíveis (ex: < 200ms).
- **ZGC / Shenandoah:** Os "ferraris" da baixa latência. Prometem pausas sub-milissegundo, mesmo em heaps de terabytes, executando quase toda a limpeza de forma concorrente.
- **Parallel GC:** Focado em throughput total (máximo processamento). Não se importa tanto com as pausas, desde que o trabalho total seja feito rápido.

## Exemplo de Monitoramento

Se você quer ver o que está acontecendo, adicione estas flags:
```bash
-Xlog:gc*:file=gc.log:time,uptime,level,tags
```
*Explicação:* Isso gera um log detalhado de cada evento de GC, quanto tempo durou a pausa STW e quanto de memória foi liberado.

## Curiosidade Técnica: O "Safepoint"

As threads não param em qualquer lugar. O GC espera que elas alcancem um **Safepoint** (geralmente em loops ou retornos de métodos). Se uma thread estiver presa em um cálculo pesado sem safepoints, ela pode atrasar o início do GC, fazendo com que todas as outras threads fiquem esperando paradas.

## Takeaway Prático: Latência vs Throughput

Ao configurar sua JVM, defina sua prioridade clara: se você opera um sistema de mensageria ou trading, utilize **ZGC** ou **Shenandoah** para manter pausas consistentes abaixo de 1ms. Se o seu foco é processamento em lote (batch) onde o tempo total de execução importa mais que interrupções momentâneas, o **Parallel GC** ainda é sua ferramenta mais eficiente. Não aceite o padrão sem antes medir sua "Stop-the-World" budget.