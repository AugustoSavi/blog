---
title: "Contrato vs. Herança: Interface e Classe Abstrata"
date: 2026-09-28 20:00:00 -0300
categories: [Java, Design Patterns]
tags: [oop, architecture, clean-code, java]
description: "Resumo de como EU escolho entre interfaces ou classes abstratas, explorando desde a semântica de design até os detalhes de bytecode."
mermaid: true
render_with_liquid: false
image:
  path: /assets/img/interface-vs-abstrata.png
  alt: "Fluxo de decisão simplificado para escolha de abstração"
---

Interface e classe abstrata nunca foram simplesmente duas formas diferentes de alcançar o mesmo resultado em Java.

Mesmo com a chegada dos `default methods` e a aproximação entre alguns de seus recursos, a escolha entre elas continua revelando decisões importantes sobre contratos, herança, estado, encapsulamento e evolução da arquitetura.

Escolher apenas com base na sintaxe pode gerar consequências que só aparecem quando o sistema começa a crescer.

## A Ilusão da Semelhança

Desde o Java 8, interfaces podem conter implementações concretas através de métodos `default`.

No Java 9, métodos `private` foram adicionados, permitindo extrair lógica auxiliar utilizada internamente por métodos `default` ou `static`.

Isso criou uma zona cinzenta:

> Se interfaces e classes abstratas podem possuir métodos com implementação, por que ainda precisamos das duas?

A resposta não está apenas no que elas **podem fazer**, mas principalmente no que elas **representam dentro do design**.

## Interfaces: O Contrato de Comportamento

Interfaces normalmente representam um **contrato, capacidade ou papel** que diferentes tipos podem assumir.

Quando uma classe implementa `Comparable`, por exemplo, ela declara que seus objetos podem participar de uma relação de ordenação.

```java
public final class Customer implements Comparable<Customer> {

    private final String name;

    public Customer(String name) {
        this.name = name;
    }

    @Override
    public int compareTo(Customer other) {
        return name.compareTo(other.name);
    }
}
```

Uma heurística bastante utilizada é pensar em interfaces como algo próximo de um **"can-do"**.

Uma classe pode:

- ser comparável;
- ser serializável;
- ser autenticável;
- executar determinada estratégia.

Essa heurística, entretanto, não é absoluta.

Interfaces como `List`, `Map`, `Set` e `CharSequence` também representam tipos importantes dentro da modelagem das APIs.

O ponto principal é que a interface define **o contrato observado externamente**, sem impor necessariamente uma hierarquia concreta de implementação.

### Características principais

- **Acoplamento mais fraco:** classes completamente diferentes podem implementar o mesmo contrato.
- **Múltiplos contratos:** uma classe pode implementar várias interfaces simultaneamente.
- **Polimorfismo:** consumidores podem depender da abstração em vez da implementação concreta.
- **Evolução de APIs:** métodos `default` permitem adicionar comportamento a interfaces existentes sem obrigar imediatamente todas as implementações a fornecer aquele método.

Esse último ponto merece cuidado.

`default methods` facilitam a evolução de APIs, mas não tornam qualquer alteração automaticamente segura. Novos métodos podem gerar conflitos entre interfaces, mudanças semânticas ou comportamento inesperado para implementações existentes.

## Classes Abstratas: A Base de uma Hierarquia

Uma classe abstrata costuma representar uma **base comum para uma família de objetos relacionados**.

Nesse caso, não estamos falando apenas sobre o que os objetos conseguem fazer, mas também sobre aspectos que fazem parte da estrutura interna daquela família:

- estado;
- invariantes;
- construção;
- implementação compartilhada;
- ciclo de vida.

Uma classe abstrata pode controlar esses elementos porque participa diretamente da hierarquia de classes.

### Características principais

- **Estado de instância:** pode possuir atributos `private`, `protected` ou com visibilidade de pacote.
- **Construtores:** pode garantir que determinado estado seja inicializado antes da criação da subclasse.
- **Herança simples:** uma classe Java pode estender apenas uma classe.
- **Métodos protegidos:** pode expor detalhes de implementação apenas para subclasses.
- **Implementação compartilhada:** pode centralizar código que faz sentido para toda a hierarquia.

A heurística clássica aqui é o **"is-a"**.

Uma implementação concreta normalmente representa uma especialização daquela classe-base.

Ainda assim, assim como ocorre com o "can-do" das interfaces, isso deve ser interpretado como uma ferramenta mental de design, e não como uma regra formal da linguagem.

## Implementação Prática e Diferenças Estruturais

Para visualizar melhor essa diferença, imagine um sistema de processamento de pagamentos.

## O Modelo com Interface

Se diferentes gateways precisam seguir o mesmo contrato, mas podem possuir implementações completamente diferentes, uma interface é uma boa escolha.

```java
import java.math.BigDecimal;

public interface PaymentGateway {

    String VERSION = "1.0.0";

    void process(BigDecimal amount);

    default void logTransaction(String id) {
        System.out.println("Logando transação: " + id);
    }
}
```

Uma implementação poderia ser:

```java
import java.math.BigDecimal;

public final class StripeGateway implements PaymentGateway {

    @Override
    public void process(BigDecimal amount) {
        System.out.println(
            "Processando " + amount + " via Stripe"
        );
    }
}
```

O consumidor pode depender apenas da abstração:

```java
public final class PaymentService {

    private final PaymentGateway gateway;

    public PaymentService(PaymentGateway gateway) {
        this.gateway = gateway;
    }

    public void pay(BigDecimal amount) {
        gateway.process(amount);
    }
}
```

Nada impede que amanhã exista:

```java
PayPalGateway
AdyenGateway
PixGateway
FakePaymentGateway
```

desde que todos respeitem o contrato definido por `PaymentGateway`.

Perceba que o serviço não precisa conhecer a estrutura interna dessas implementações.

## O Modelo com Classe Abstrata

Agora imagine que determinados gateways remotos compartilhem:

- chave de API;
- timeout;
- validação;
- coleta de métricas;
- fluxo de execução.

Nesse cenário, uma classe abstrata pode representar melhor essa infraestrutura comum.

```java
import java.math.BigDecimal;
import java.time.Duration;
import java.util.Objects;

public abstract class BaseRemoteGateway {

    private final String apiKey;
    private final Duration connectionTimeout;

    protected BaseRemoteGateway(
        String apiKey,
        Duration connectionTimeout
    ) {
        this.apiKey = Objects.requireNonNull(
            apiKey,
            "apiKey não pode ser nula"
        );

        this.connectionTimeout = Objects.requireNonNull(
            connectionTimeout,
            "connectionTimeout não pode ser nulo"
        );

        if (apiKey.isBlank()) {
            throw new IllegalArgumentException(
                "apiKey não pode ser vazia"
            );
        }
    }

    public final void execute(BigDecimal amount) {
        validateConnection();
        doProcess(amount);
        recordMetrics();
    }

    protected abstract void doProcess(BigDecimal amount);

    protected final String apiKey() {
        return apiKey;
    }

    protected final Duration connectionTimeout() {
        return connectionTimeout;
    }

    private void validateConnection() {
        System.out.println("Validando conexão...");
    }

    private void recordMetrics() {
        System.out.println(
            "Métrica enviada para o gateway."
        );
    }
}
```

Uma implementação concreta pode customizar apenas o ponto necessário:

```java
import java.math.BigDecimal;
import java.time.Duration;

public final class StripeRemoteGateway
        extends BaseRemoteGateway {

    public StripeRemoteGateway(String apiKey) {
        super(
            apiKey,
            Duration.ofSeconds(5)
        );
    }

    @Override
    protected void doProcess(BigDecimal amount) {
        System.out.println(
            "Executando pagamento no Stripe: " + amount
        );
    }
}
```

Nesse exemplo, a classe abstrata controla o fluxo:

```text
validar conexão
      ↓
processar pagamento
      ↓
registrar métricas
```

Enquanto a subclasse implementa apenas a etapa variável.

Essa estrutura é um exemplo clássico do padrão **Template Method**.

## Por Que `execute()` é `final`?

Existe um detalhe importante no exemplo:

```java
public final void execute(BigDecimal amount)
```

O método é `final` propositalmente.

A intenção da classe-base é garantir que o fluxo seja sempre:

```text
validateConnection()
doProcess()
recordMetrics()
```

A subclasse pode customizar `doProcess()`, mas não pode alterar arbitrariamente a ordem do algoritmo sobrescrevendo `execute()`.

Esse tipo de controle é uma das situações em que classes abstratas continuam extremamente úteis.

## Interfaces Não Possuem Estado de Instância

Uma diferença estrutural importante é que interfaces não possuem estado associado a cada objeto.

Quando declaramos:

```java
interface PaymentGateway {

    String VERSION = "1.0.0";
}
```

esse campo é implicitamente:

```java
public static final String VERSION = "1.0.0";
```

Ou seja, pertence à interface e não a cada instância.

Uma interface não pode declarar algo equivalente a:

```java
private String apiKey;
```

como estado individual de cada implementação.

Cada classe concreta é responsável por armazenar seu próprio estado.

Classes abstratas, por outro lado, podem possuir estado normalmente.

## Construtores

Classes abstratas também podem possuir construtores.

```java
protected BaseRemoteGateway(String apiKey) {
    this.apiKey = apiKey;
}
```

Ao construir uma subclasse:

```java
new StripeRemoteGateway("secret");
```

o construtor da superclasse participa da cadeia de inicialização através de `super(...)`.

Isso permite que a classe-base estabeleça invariantes antes que o objeto esteja completamente construído.

Interfaces não possuem construtores porque não representam instâncias independentes nem armazenam estado de instância.

## Métodos `default`

Desde o Java 8, interfaces podem fornecer implementação através de métodos `default`.

```java
public interface PaymentGateway {

    void process(BigDecimal amount);

    default void validateAmount(BigDecimal amount) {
        if (amount.signum() <= 0) {
            throw new IllegalArgumentException(
                "O valor deve ser positivo."
            );
        }
    }
}
```

O objetivo não foi transformar interfaces em classes abstratas.

Uma das principais motivações foi permitir a evolução de APIs existentes.

Imagine uma interface utilizada por milhares de implementações:

```java
interface CollectionLike {

    void add(Object value);
}
```

Adicionar diretamente um novo método abstrato:

```java
void sort();
```

obrigaria todas as implementações existentes a implementá-lo.

Com `default`, a API pode fornecer um comportamento padrão:

```java
default void sort() {
    // implementação padrão
}
```

Isso reduz o impacto de determinadas evoluções no contrato.

## Métodos `private` em Interfaces

Desde o Java 9, interfaces também podem possuir métodos `private`.

```java
public interface PaymentGateway {

    default void execute() {
        validate();
        process();
    }

    private void validate() {
        System.out.println("Validando...");
    }

    private void process() {
        System.out.println("Processando...");
    }
}
```

Esses métodos servem principalmente para reutilizar lógica interna entre métodos `default` ou `static`.

Eles não ficam disponíveis para as classes que implementam a interface.

Portanto:

```java
gateway.validate();
```

não seria permitido fora da própria interface.

## O Funcionamento Interno: Bytecode

As diferenças entre interfaces e classes também aparecem no bytecode gerado pelo compilador Java.

Considere:

```java
PaymentGateway gateway = new StripeGateway();

gateway.process(
    new BigDecimal("100.00")
);
```

Como o tipo estático da referência é `PaymentGateway`, a chamada normalmente será representada no bytecode pela instrução:

```text
invokeinterface
```

Já considere:

```java
StripeGateway gateway = new StripeGateway();

gateway.process(
    new BigDecimal("100.00")
);
```

Nesse caso, uma chamada de método de instância comum utiliza normalmente:

```text
invokevirtual
```

A JVM possui instruções distintas porque a resolução dos métodos de classes e interfaces segue regras diferentes.

Isso pode ser observado facilmente utilizando:

```bash
javac Main.java
javap -c Main.class
```

Um bytecode simplificado poderia mostrar algo semelhante a:

```text
invokeinterface PaymentGateway.process
```

ou:

```text
invokevirtual StripeGateway.process
```

dependendo do tipo da referência e da chamada realizada.

## `invokevirtual` vs. `invokeinterface`: Existe Diferença de Performance?

Historicamente, implementações de JVM precisavam utilizar estruturas diferentes para despachar chamadas de métodos de classes e interfaces.

É comum encontrar explicações baseadas em estruturas conceitualmente semelhantes a:

```text
vtable
```

para classes, e:

```text
itable
```

para interfaces.

Entretanto, isso é um detalhe de implementação da JVM, e não uma característica que deve orientar decisões arquiteturais.

JVMs modernas como a HotSpot possuem otimizações como:

- profiling;
- JIT compilation;
- method inlining;
- devirtualization;
- inline caches;
- speculative optimization.

Em muitos caminhos quentes de execução, o JIT consegue determinar qual implementação será chamada e pode até eliminar completamente o custo do despacho virtual.

Por isso:

> Escolher uma classe abstrata em vez de uma interface por suposta vantagem de `invokevirtual` sobre `invokeinterface` quase nunca faz sentido.

A distinção é muito mais interessante para entender o funcionamento da JVM do que para tomar uma decisão de design.

## Um Exemplo Curioso com Bytecode

Considere:

```java
public interface Printer {

    void print();
}
```

e:

```java
public final class ConsolePrinter implements Printer {

    @Override
    public void print() {
        System.out.println("Hello");
    }
}
```

Agora:

```java
public final class Main {

    public static void main(String[] args) {

        Printer printer = new ConsolePrinter();

        printer.print();
    }
}
```

Ao executar:

```bash
javac Main.java Printer.java ConsolePrinter.java
javap -c Main
```

podemos encontrar algo semelhante a:

```text
invokeinterface #...
```

Mesmo sabendo em tempo de compilação que o objeto criado foi um `ConsolePrinter`.

Isso acontece porque a instrução é determinada a partir da chamada no bytecode e do tipo utilizado naquela operação.

Em runtime, entretanto, o JIT pode perceber que aquele ponto é monomórfico e realizar otimizações muito mais agressivas.

É justamente por isso que olhar apenas para bytecode não é suficiente para inferir a performance final de um programa Java.

## Padrões de Projeto Associados

A escolha entre interfaces e classes abstratas aparece naturalmente em diversos padrões de projeto.

## Strategy Pattern

O Strategy Pattern frequentemente utiliza interfaces.

```java
public interface PricingStrategy {

    BigDecimal calculate(BigDecimal amount);
}
```

Implementações diferentes:

```java
public final class RegularPricing
        implements PricingStrategy {

    @Override
    public BigDecimal calculate(BigDecimal amount) {
        return amount;
    }
}
```

```java
public final class DiscountPricing
        implements PricingStrategy {

    @Override
    public BigDecimal calculate(BigDecimal amount) {
        return amount.multiply(
            new BigDecimal("0.90")
        );
    }
}
```

O consumidor conhece apenas:

```java
PricingStrategy
```

e pode trocar a implementação livremente.

### Strategy com Lambdas

Em Java moderno, Strategies pequenas podem inclusive ser representadas por interfaces funcionais.

```java
@FunctionalInterface
public interface PricingStrategy {

    BigDecimal calculate(BigDecimal amount);
}
```

Permitindo:

```java
PricingStrategy tenPercentDiscount =
    amount -> amount.multiply(
        new BigDecimal("0.90")
    );
```

Essa é uma vantagem importante das interfaces dentro do ecossistema Java moderno.

## Template Method Pattern

O Template Method normalmente utiliza herança.

Uma classe-base define o algoritmo:

```java
public abstract class PaymentProcessor {

    public final void process() {
        validate();
        authorize();
        persist();
    }

    protected abstract void authorize();

    private void validate() {
        System.out.println("Validando...");
    }

    private void persist() {
        System.out.println("Persistindo...");
    }
}
```

A subclasse fornece apenas uma etapa:

```java
public final class PixPaymentProcessor
        extends PaymentProcessor {

    @Override
    protected void authorize() {
        System.out.println(
            "Autorizando pagamento PIX..."
        );
    }
}
```

A diferença conceitual fica clara:

```text
Strategy
    ↓
composição

Template Method
    ↓
herança
```

## Composição Antes de Herança

Um erro comum é utilizar uma classe abstrata apenas para reutilizar código.

Por exemplo:

```java
abstract class BaseService {

    protected void log() {
        ...
    }
}
```

e então várias classes passam a estendê-la somente porque precisam de `log()`.

Nesse caso, a herança está sendo utilizada apenas como mecanismo de reutilização.

Frequentemente seria melhor utilizar composição:

```java
public final class Logger {

    public void log() {
        ...
    }
}
```

e então:

```java
public final class PaymentService {

    private final Logger logger;

    public PaymentService(Logger logger) {
        this.logger = logger;
    }
}
```

Herança cria uma relação estrutural forte.

Composição normalmente oferece mais flexibilidade.

Uma boa regra é:

> Não use uma classe abstrata apenas porque você quer evitar duplicação de código.

Herança deve representar uma relação coerente dentro do modelo.

## Interface + Implementação Esqueleto

Existe uma abordagem particularmente interessante no Java:

```text
Interface
    +
Classe abstrata opcional
```

A interface define o contrato público.

```java
public interface MyList<E> {

    E get(int index);

    int size();
}
```

Uma classe abstrata pode fornecer uma implementação parcial:

```java
public abstract class AbstractMyList<E>
        implements MyList<E> {

    public boolean isEmpty() {
        return size() == 0;
    }
}
```

O desenvolvedor então possui duas opções.

Pode implementar diretamente:

```java
class CustomList<E> implements MyList<E> {
    ...
}
```

ou aproveitar o esqueleto:

```java
class CustomList<E>
        extends AbstractMyList<E> {
    ...
}
```

Esse padrão é conhecido como **Skeletal Implementation**.

A Java Collections Framework utiliza essa estratégia extensivamente.

Um dos exemplos mais conhecidos é:

```java
List<E>
```

com:

```java
AbstractList<E>
```

A interface define o contrato.

A classe abstrata oferece uma implementação base para quem deseja aproveitá-la.

Esse modelo consegue combinar duas propriedades importantes:

```text
baixo acoplamento no contrato
        +
reutilização opcional de implementação
```

## Quando Usar Cada Um?

Uma forma prática de tomar a decisão é perguntar:

> Estou tentando definir **o que um objeto deve fazer** ou controlar **como uma família de objetos funciona**?

Se o foco está no contrato, uma interface tende a ser a escolha inicial.

Use **interfaces** quando:

- diferentes classes precisam obedecer ao mesmo contrato;
- as implementações podem não possuir qualquer relação entre si;
- você deseja favorecer composição;
- você precisa permitir múltiplos papéis;
- você quer facilitar substituição de implementações;
- consumidores devem depender apenas da abstração;
- a abstração pode ser representada por uma interface funcional.

Use **classes abstratas** quando:

- existe estado comum entre as subclasses;
- existe um processo de inicialização compartilhado;
- construtores precisam garantir invariantes;
- existe código comum fortemente ligado à hierarquia;
- métodos `protected` fazem sentido;
- existe um algoritmo-base que subclasses devem completar;
- a relação de herança representa corretamente o modelo.

## Um Fluxo Mental Simples

Eu costumo pensar na decisão aproximadamente desta forma:

```text
Preciso apenas definir um contrato?
          |
         Sim
          |
      Interface
          |
          v
Existe implementação comum opcional?
          |
         Sim
          |
Interface + Classe Abstrata Esqueleto
```

Caso o problema já comece com:

```text
estado compartilhado
+
construtores
+
invariantes
+
algoritmo-base
```

uma classe abstrata pode ser um ponto de partida mais natural.

## Regra que Eu Uso

Minha regra prática é:

> Comece pensando no contrato antes de pensar na herança.

Se diferentes implementações conseguem obedecer à mesma abstração sem compartilhar uma identidade estrutural, normalmente começo com uma interface.

Se percebo que existe uma verdadeira família de objetos compartilhando:

- estado;
- invariantes;
- comportamento;
- ciclo de vida;

então considero uma classe abstrata.

E, quando quero os benefícios das duas abordagens, utilizo:

```text
Interface pública
       +
Classe abstrata opcional
```

Esse modelo mantém os consumidores desacoplados da hierarquia enquanto ainda permite reutilização para implementações que desejarem utilizá-la.

No fim, a diferença entre interface e classe abstrata não está simplesmente em qual delas possui métodos com corpo.

A verdadeira decisão é entre:

```text
Contrato
vs.
Hierarquia
```

E entender essa distinção é muito mais importante para a arquitetura do sistema do que qualquer diferença sintática entre `interface` e `abstract class`.

---

### Referências

- [Java Language Specification — Interfaces](https://docs.oracle.com/javase/specs/jls/se25/html/jls-9.html)
- [Java Virtual Machine Specification — Instruction Set](https://docs.oracle.com/javase/specs/jvms/se25/html/jvms-6.html)
- [Oracle Java Documentation — Default Methods](https://docs.oracle.com/javase/tutorial/java/IandI/defaultmethods.html)
- [Oracle Java Documentation — Abstract Methods and Classes](https://docs.oracle.com/javase/tutorial/java/IandI/abstract.html)
- [Java API — AbstractList](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/AbstractList.html)