---
title: "Padrão Strategy"
date: 2026-01-21 09:00:00 -0300
categories: [Design Patterns, Java]
tags: [design patterns, java]
render_with_liquid: false
---

Você abre um arquivo de serviço e encontra um método com 300 linhas, onde metade delas são blocos `if-else` ou `switch-case` verificando tipos de pagamento, tipos de usuário ou planos de assinatura. Este é o famoso "Código Espaguete". O padrão **Strategy** é a cura definitiva para esse sintoma.

## O Princípio Open/Closed

O padrão Strategy permite que você defina uma família de algoritmos, coloque cada um deles em uma classe separada e torne seus objetos intercambiáveis. Isso segue o "O" do SOLID: seu código fica aberto para extensão (novas estratégias) mas fechado para modificação (você não mexe no código que já funciona).

## Código sujo

```java
public void processPayment(String type, double amount) {
    if (type.equals("CREDIT_CARD")) {
        // 20 linhas de lógica
    } else if (type.equals("PIX")) {
        // 15 linhas de lógica
    } else if (type.equals("BOLETO")) {
        // 30 linhas de lógica
    }
}
```

## A Solução (O Código 'Elegante')

1.  **Crie a Interface:**
```java
public interface PaymentStrategy {
    void process(double amount);
    boolean appliesTo(String type);
}
```

2.  **Implemente as Estratégias:**
```java
@Component
public class PixPayment implements PaymentStrategy {
    public void process(double amount) { /* lógica Pix */ }
    public boolean appliesTo(String type) { return type.equals("PIX"); }
}
```

3.  **O Contexto (Service):**
```java
@Service
public class PaymentService {
    private final List<PaymentStrategy> strategies; // O Spring injeta todas automaticamente!

    public void process(String type, double amount) {
        strategies.stream()
            .filter(s -> s.appliesTo(type))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("Tipo inválido"))
            .process(amount);
    }
}
```

## Por que isso é incrível?

1.  **Testabilidade:** Você pode testar cada estratégia isoladamente.
2.  **SRP (Single Responsibility):** Cada classe só cuida de um tipo de processamento.
3.  **Facilidade de Extensão:** Quer adicionar "Cripto"? Basta criar uma nova classe que implementa a interface. Você não toca no `PaymentService`.

## Dica: Strategy + Factory

Para evitar percorrer a lista em todo processamento, você pode criar uma `Factory` que mapeia o tipo para a estratégia em um `Map` no momento do startup da aplicação.

## O Princípio da Delegação de Comportamento

A regra de ouro para um código extensível é: se você tem uma lógica que varia com base em uma condição, essa lógica não pertence ao seu serviço principal, mas sim a uma estratégia especializada. O padrão Strategy nos ensina que o papel do orquestrador é saber **quem** deve fazer o trabalho, e não **como** o trabalho deve ser feito. Ao delegar o comportamento para classes isoladas, você transforma condicionais rígidas em um sistema dinâmico e plugável.