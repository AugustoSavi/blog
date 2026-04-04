---
title: "Por que x++ == ++x é false no Java?"
date: 2026-04-02 09:00:00 -0300
categories: [Java, Certificacao]
tags: [java, jvm, bytecode, increment]
render_with_liquid: false
mermaid: true
---

Estou iniciando uma nova jornada que é (tentar 🙂) tirar uma certificação Java. 

Este post marca o início de uma série onde vou estar gerando conteudo com as coisas que acho legal trazer sobre o processo...

Para começar, vamos analisar uma expressão que já vi em alguns simulados: `x++ == ++x`. Se você olhar rápido, pode pensar que, como ambos incrementam a mesma variável, o resultado deveria ser `true`, mas não é o que ocorre... 

## O que ocorre

O segredo não está apenas no valor final da variável, mas em **quando** o valor é entregue para a expressão que o utiliza.

```java
class ComparacaoIncrement {
    void main() {
        int x = 69;
        IO.println(x++ == ++x); // false
        IO.println(x); // 71
    }
}
```
{: file="ComparacaoIncrement.java" }

A execução resulta em `false` e o valor final de `x` é `71`. Para entender o porquê, precisamos olhar a sequência exata de operações na **Pilha de Operandos (Operand Stack)**.

### Passo a Passo no Bytecode

Para ver bytecode estou usando: 

```bash
javac ComparacaoIncrement.java && javap -c -v ComparacaoIncrement > ComparacaoIncrement.bytecode
```

Isso mostra:

- Constant Pool
- Stack size
- Local variables
- Métodos detalhados


Aqui está o bytecode detalhado do método `main`:

```java
  void main();
    Code:
       0: bipush        69        // 1. Coloca 69 na Stack
       2: istore_1                // 2. Armazena na variável local 1 (x = 69)
       3: iload_1                 // 3. Carrega o valor ATUAL (69) na Stack (para o lado esquerdo do ==)
       4: iinc          1, 1      // 4. Incrementa x na memória (x agora é 70)
       7: iinc          1, 1      // 5. Incrementa x na memória novamente (x agora é 71)
      10: iload_1                 // 6. Carrega o valor ATUAL (71) na Stack (para o lado direito do ==)
      11: if_icmpne     18        // 7. Compara os valores no topo da Stack: 69 == 71?
```
{: file="ComparacaoIncrement.bytecode" }

A instrução `iinc` é única porque ela modifica a variável local diretamente na memória **sem passar pela pilha de operandos**. Isso cria o cenário onde o valor "antigo" já está na pilha esperando a comparação, enquanto a memória já foi atualizada.

### Visualizando a Pilha

O diagrama abaixo ilustra o estado da Stack e da Memória (Variáveis Locais) durante a execução da linha crítica:

```mermaid
flowchart LR
    %% ===== PASSO 3 =====
    subgraph P3 ["Step 3"]
        direction TB
        S1["Stack<br/>69"]
        M1["Memory (x)<br/>69"]
    end

    %% ===== PASSO 4 =====
    subgraph P4 ["Step 4"]
        direction TB
        S2["Stack<br/>69"]
        M2["Memory (x)<br/>70"]
    end

    %% ===== PASSO 5 =====
    subgraph P5 ["Step 5"]
        direction TB
        S3["Stack<br/>69"]
        M3["Memory (x)<br/>71"]
    end

    %% ===== PASSO 6 =====
    subgraph P6 ["Step 6"]
        direction TB
        S4["Stack<br/>69, 71"]
        M4["Memory (x)<br/>71"]
    end

    %% ===== FLOW =====
    P3 --> P4 --> P5 --> P6

    %% ===== ALIGNMENT (invisible links) =====
    S1 --- M1
    S2 --- M2
    S3 --- M3
    S4 --- M4
```

O resultado é `false` porque o Java avalia expressões da esquerda para a direita. 

1.  O `x++` avalia para `69` e coloca isso na pilha.
2.  Como efeito colateral, `x` vira `70`.
3.  Em seguida, o `++x` incrementa `x` para `71` e coloca `71` na pilha.
4.  A comparação final é `69 == 71`.

> Lembre-se que o operador de igualdade `==` não é um ponto de sincronização. Ele apenas compara os valores que já foram resolvidos e colocados na pilha de operandos seguindo a precedência e a ordem de avaliação (esquerda para a direita).
{: .prompt-tip }

Entender o bytecode não serve apenas para passar em provas, mas para compreender como o código que escrevemos é realmente interpretado pela máquina que o executa.


## Referencias

- [javainuse](https://www.javainuse.com/quiz/1z08291)


## Enviroment

```bash
$ java -version
openjdk version "25.0.2" 2026-01-20
OpenJDK Runtime Environment (build 25.0.2+10-Ubuntu-124.04)
OpenJDK 64-Bit Server VM (build 25.0.2+10-Ubuntu-124.04, mixed mode, sharing)
```