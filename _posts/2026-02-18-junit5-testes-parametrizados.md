---
title: "JUnit 5 e Testes Parametrizados: Testando Benefícios com Elegância"
date: 2026-02-18 09:00:00 -0300
categories: [Testes, JUnit 5]
tags: [testes, junit 5]
render_with_liquid: false
---

Em sistemas de benefícios, uma transação pode passar ou falhar por dezenas de motivos: saldo insuficiente, categoria errada, cartão bloqueado, limite diário excedido, etc. Se você escrever um método de teste para cada cenário, seu arquivo de teste terá 2000 linhas. O **@ParameterizedTest** do JUnit 5 é o seu melhor amigo.

## O Código Repetitivo

Você já se viu copiando e colando o mesmo teste, mudando apenas um valor e o resultado esperado? Isso é um sinal de que você deve usar testes parametrizados.

## Exemplo Prático: Validação de Categorias

Imagine que queremos testar se o cartão de benefícios aceita o estabelecimento baseado no código de categoria (MCC).

```java
class MerchantValidationTest {

    @ParameterizedTest
    @CsvSource({
        "5812, MEAL, true",    // Restaurante no benefício Refeição
        "5411, FOOD, true",    // Supermercado no benefício Alimentação
        "5812, FOOD, false",   // Restaurante NÃO passa no Alimentação pura
        "5000, MEAL, false"    // Loja de Ferragens NÃO passa no Refeição
    })
    void shouldValidateMerchantCategory(int mcc, Category category, boolean expectedResult) {
        boolean result = validator.isValid(mcc, category);
        assertEquals(expectedResult, result);
    }
}
```

## Fontes de Dados (Sources)

O JUnit 5 oferece várias formas de injetar dados:
- **@ValueSource:** Para uma lista simples de strings ou números.
- **@CsvSource:** Para múltiplos parâmetros (como o exemplo acima).
- **@MethodSource:** Para cenários complexos onde você precisa de uma função que retorne uma lista de objetos.

## Por que isso é bom?

1.  **Manutenibilidade:** Se a regra de negócio mudar, você altera apenas uma linha na tabela de dados, não o código do teste.
2.  **Cobertura:** Facilita testar "Edge Cases" (casos de borda) que você poderia esquecer se tivesse que escrever um teste completo para cada um.
3.  **Legibilidade:** O nome do teste no relatório fica claro: `shouldValidateMerchantCategory[1] mcc=5812, category=MEAL -> true`.

## Conclusão

Testes parametrizados tornam sua suíte de testes mais inteligente e compacta. Em sistemas com regras complexas, eles são essenciais para garantir que todas as variações de benefícios sejam validadas sem criar um código de teste impossível de ler.