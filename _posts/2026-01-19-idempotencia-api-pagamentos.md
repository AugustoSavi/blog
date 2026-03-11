---
title: "Idempotência em APIs"
date: 2026-01-19 09:00:00 -0300
categories: [API Design, Sistemas Distribuídos]
tags: [api design, sistemas distribuídos]
render_with_liquid: false
---

Imagine que você está pagando um boleto no seu app. Você clica em "Confirmar", a internet oscila e o app mostra um erro. Você clica de novo. Sem **Idempotência**, você acabaria pagando o boleto duas vezes. Em sistemas financeiros, isso é inaceitável.

## Rede Não Confiável

Em sistemas distribuídos, existem três tipos de falhas:
1.  A requisição não chegou ao servidor.
2.  O servidor processou, mas a resposta não chegou ao cliente.
3.  O servidor travou no meio do processamento.

A Idempotência garante que, não importa quantas vezes você envie a mesma requisição, o efeito colateral no servidor aconteça **apenas uma vez**.

## Como Implementar: A Chave de Idempotência

A forma mais comum é usar um Header chamado `Idempotency-Key` (um UUID gerado pelo cliente).

**O Fluxo no Servidor:**
1.  Recebe a requisição e a `Idempotency-Key`.
2.  Verifica no banco (ou Redis) se aquela chave já foi processada.
3.  **Se já foi:** Retorna o resultado salvo anteriormente (sem processar de novo).
4.  **Se não foi:** Processa, salva o resultado associado à chave e retorna.

## Exemplo de Lógica

```java
@PostMapping("/payments")
public ResponseEntity<PaymentResponse> createPayment(
    @RequestHeader("Idempotency-Key") String key,
    @RequestBody PaymentRequest request) {
    
    Optional<PaymentResponse> cached = cacheService.get(key);
    if (cached.isPresent()) {
        return ResponseEntity.ok(cached.get()); // Retorna o que já foi feito
    }

    PaymentResponse response = paymentService.process(request);
    cacheService.put(key, response);
    
    return ResponseEntity.status(201).body(response);
}
```

## Verbos HTTP e Idempotência Nativa

De acordo com a especificação HTTP:
- **GET, PUT, DELETE:** Devem ser idempotentes por definição.
- **POST:** **Não** é idempotente por definição. É aqui que precisamos da nossa lógica customizada.

## Curiosidade: Janela de Idempotência

Você não precisa guardar as chaves para sempre. Geralmente, chaves de idempotência são mantidas por 24 a 48 horas, tempo suficiente para o cliente tentar novamente em caso de falha de rede.

## O Princípio da Repetibilidade Segura

Em sistemas distribuídos, uma operação só é considerada completa quando o cliente recebe a confirmação, e não apenas quando o servidor processa o dado. A regra fundamental é: **toda operação de escrita (POST) deve ser tratada como potencialmente repetida**. Implementar a idempotência através de chaves únicas geradas pelo cliente é o único contrato que garante a integridade do estado global diante da inevitável instabilidade das redes.