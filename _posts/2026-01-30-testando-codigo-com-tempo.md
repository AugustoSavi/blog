---
title: "Como testar código que depende do Tempo (sem ficar maluco)"
date: 2026-01-30 09:00:00 -0300
categories: [Java, Testes]
tags: [java, testes]
render_with_liquid: false
---

Testar lógica que depende da hora atual (`LocalDateTime.now()` ou `System.currentTimeMillis()`) é um pesadelo. Se o seu teste roda às 23:59 e termina às 00:01, ele pode falhar aleatoriamente. Como tornar o tempo determinístico nos seus testes?

## O Inimigo do Determinismo

Um bom teste deve ser determinístico: rodar 1000 vezes e passar 1000 vezes. O tempo do sistema é o oposto disso; ele está sempre mudando. Se você tem uma regra de "desconto para compras feitas após as 18h", você não pode esperar dar 18h para rodar o teste.

## O Jeito Errado (Hardcoded)

```java
public void process() {
    if (LocalDateTime.now().getHour() > 18) { // Jamais faça isso!
        applyDiscount();
    }
}
```
*Problema:* Este código é impossível de testar de forma confiável sem mudar o relógio do sistema operacional (o que é uma ideia terrível).

## O Jeito Certo: Abstraia o Relógio (Java 8+ Clock)

O Java 8 introduziu a classe `java.time.Clock`. Em vez de chamar o tempo estático, você pede o tempo para o relógio.

1.  **Injete o Clock no seu Serviço:**
```java
@Service
public class DiscountService {
    private final Clock clock;

    public DiscountService(Clock clock) {
        this.clock = clock;
    }

    public void process() {
        if (LocalDateTime.now(clock).getHour() > 18) {
            applyDiscount();
        }
    }
}
```

2.  **No Código de Produção:** Injete o `Clock.systemDefaultZone()`.

3.  **No Código de Teste:** Injete um **Fixed Clock**.
```java
@Test
void shouldApplyDiscountAfter18h() {
    // Congela o tempo às 19:00 de um dia específico
    Instant fixedInstant = Instant.parse("2024-03-10T19:00:00Z");
    Clock fixedClock = Clock.fixed(fixedInstant, ZoneId.of("UTC"));
    
    DiscountService service = new DiscountService(fixedClock);
    service.process();
    
    assertTrue(discountApplied);
}
```

## Vantagens dessa abordagem

- **Determinismo total:** O tempo nunca muda durante o teste.
- **Simulação de cenários:** Você pode testar anos bissextos, viradas de mês ou horários de verão apenas mudando o `fixedInstant`.
- **Sem Mocks complexos:** Você usa uma classe real do Java (`Clock.fixed`) em vez de tentar mockar métodos estáticos com bibliotecas pesadas.

## Curiosidade: Outras Linguagens

Quase todas as linguagens modernas seguem esse padrão. No mundo Node/JS, usamos bibliotecas como `sinon` para fazer o "fake timers". No Go, passamos uma interface de `TimeProvider`.

## Conclusão

Nunca dependa do relógio do sistema diretamente. Ao injetar o `Clock`, você ganha o controle total sobre a quarta dimensão e garante que seus testes sejam rápidos, confiáveis e independentes do momento em que são executados.