---
title: "Como dimensionar recursos da JVM sem adivinhação"
date: 2026-03-10 09:00:00 -0300
categories: [JVM, Performance]
tags: [java, jvm, performance, devops, monitoramento]
render_with_liquid: false
---

Você já passou pela situação de ver sua aplicação Java consumir 8GB de RAM em um container enquanto o Heap estava configurado para apenas 4GB? Ou pior: sua aplicação sofre com pausas longas de Garbage Collection e a solução do time foi simplesmente "dar mais memória"?

Muitas vezes, o dimensionamento de recursos da JVM é tratado como alquimia ou tentativa e erro. No entanto, existe uma ciência por trás de como a Java Virtual Machine utiliza a memória do sistema operacional e como você pode calcular os limites ideais para evitar o temido OOM (Out Of Memory) Killer do Linux.

## O Mito do "Quanto mais, melhor"

Existe uma crença comum de que memória sobrando é sempre bom. Na JVM, isso é parcialmente falso. Um Heap excessivamente grande pode resultar em pausas de Garbage Collection muito mais longas (Stop-the-World), pois o GC terá que varrer um oceano de objetos para decidir o que limpar. 

O objetivo não é ter memória sobrando, mas sim ter o **equilíbrio** entre throughput e latência.

## Anatomia da Memória Além do Heap

Para dimensionar corretamente, você precisa entender que o processo Java consome mais do que apenas o Heap. A memória total (RSS - Resident Set Size) é composta por:

1.  **Heap:** Onde moram seus objetos.
2.  **Metaspace:** Onde a JVM armazena metadados das classes.
3.  **Code Cache:** Memória para o JIT Compiler armazenar código compilado.
4.  **Stack:** Memória para cada thread (geralmente 1MB por thread).
5.  **Direct Buffers:** Memória usada para I/O de alta performance (NIO).
6.  **Native Memory:** Memória usada pela própria JVM e bibliotecas nativas.

> Se você configurar um container com 2GB de RAM e definir o `-Xmx` (Heap máximo) como 2GB, seu processo será morto pelo sistema operacional quase instantaneamente.
{: .prompt-warning }

## Ferramentas de Monitoramento

Antes de ajustar, você deve observar. Aqui estão as ferramentas essenciais:

### 1. VisualVM ou JConsole
Excelentes para ambiente de desenvolvimento. Elas mostram em tempo real o crescimento de cada região da memória.

### 2. Native Memory Tracking (NMT)
Esta é uma flag interna da JVM que permite ver exatamente onde a memória "não-heap" está sendo gasta.

```bash
# Habilite o NMT na inicialização
-XX:NativeMemoryTracking=summary
```

Depois, você pode consultar os dados via `jcmd`:
```bash
jcmd <pid> VM.native_memory summary
```

### 3. Micrometer + Prometheus + Grafana
O padrão ouro para produção. Através do Micrometer (comum no Spring Boot), você expõe métricas como `jvm.memory.used` e `jvm.gc.pause`.

## A Fórmula Prática de Dimensionamento

Uma regra de ouro para aplicações em containers (Docker/Kubernetes) é reservar cerca de **25% a 30%** da memória do container para a "Overhead" da JVM (não-heap) e o restante para o Heap.

### Exemplo de Configuração Ideal

Se você tem um limite de **1GB** no seu container:

```bash
java -Xms640m -Xmx640m \
     -XX:MaxMetaspaceSize=128m \
     -XX:ReservedCodeCacheSize=64m \
     -Xss512k \
     -jar app.jar
```

**Explicação dos parâmetros:**
- `-Xms` e `-Xmx`: Definidos com o mesmo valor para evitar o custo de redimensionamento do Heap em runtime.
- `-XX:MaxMetaspaceSize`: Evita que vazamentos de classes (comuns em frameworks que geram proxies) consumam toda a RAM nativa.
- `-Xss512k`: Reduzi o tamanho da stack de cada thread de 1MB para 512KB, economizando memória se você tiver muitas threads.

## Funcionamento Interno: Por que igualar Xms e Xmx?

Quando você define `-Xms` menor que `-Xmx`, a JVM tenta ser "econômica". Quando o Heap enche, ela dispara um Garbage Collection Full para tentar liberar espaço antes de pedir mais memória ao Sistema Operacional. 

Se você já sabe que sua aplicação vai precisar de 1GB, definir `Xms` e `Xmx` iguais evita essas pausas de GC desnecessárias e o overhead de alocação dinâmica de memória nativa.

## Curiosidade Técnica: Compressed OOPs

Você sabia que em Heaps de até **32GB**, a JVM usa uma técnica chamada *Compressed Ordinary Object Pointers*? Ela armazena referências de 64 bits em apenas 32 bits, economizando um espaço massivo. Se você aumentar o seu Heap de 31GB para 33GB, sua aplicação pode acabar tendo **menos** memória efetiva disponível para objetos, pois todas as referências dobrarão de tamanho!

## Aplicações Práticas: O cenário "Cloud Native"

No Kubernetes, utilize sempre as novas flags de percepção de container:

```bash
-XX:+UseContainerSupport \
-XX:MaxRAMPercentage=75.0
```

Essas flags dizem à JVM para olhar o limite do cgroup (o limite do container) em vez da memória total do servidor físico. Definir `MaxRAMPercentage=75.0` é uma forma dinâmica e segura de garantir que o Heap nunca sufoque o restante do sistema.

## Conclusão

Dimensionar a JVM não é apenas configurar o `-Xmx`. É entender o consumo holístico de memória do processo Java. Comece monitorando o NMT e as métricas de Metaspace, e sempre deixe uma margem de manobra para a memória nativa. No mundo de microserviços, ser preciso no uso de recursos é a diferença entre uma infraestrutura barata e estável ou uma cheia de instabilidades inexplicáveis.
