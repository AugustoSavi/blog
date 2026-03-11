---
title: "Transações e APIs Externas"
date: 2026-01-14 09:00:00 -0300
categories: [Java, Spring]
tags: [java, spring]
render_with_liquid: false
---

Você tem um método anotado com `@Transactional`. Dentro dele, você atualiza o banco, chama uma API externa (ex: gateway de pagamento) e depois atualiza o banco novamente. Parece correto? Na verdade, essa é uma das receitas mais perigosas para derrubar seu sistema em produção.

## O Sequestro do Pool de Conexões

O problema não é funcional, é de infraestrutura. Quando você entra em um método `@Transactional`, o Hibernate reserva uma conexão do seu **Pool de Conexões (HikariCP)** e a mantém presa até o método terminar.

## O Cenário do Desastre

Se a sua API externa demorar 10 segundos para responder (por lentidão na rede ou no parceiro), aquela conexão do banco ficará **parada, sem fazer nada**, por 10 segundos. Se você tiver 100 requisições simultâneas fazendo isso e seu pool tiver 100 conexões, seu sistema inteiro para. Ninguém mais consegue fazer nem um `SELECT` simples, porque todas as conexões estão "sequestradas" esperando a API externa.


**Nunca, jamais, mantenha uma transação de banco aberta durante uma chamada de rede lenta.**

## Como resolver: O Padrão de Três Passos

Em vez de uma transação gigante, divida o trabalho:

1.  **Pré-Processamento:** Salve a intenção no banco (ex: Status "PENDING") e **feche a transação**.
2.  **Chamada Externa:** Chame a API externa fora de qualquer contexto `@Transactional`.
3.  **Pós-Processamento:** Abra uma nova transação para atualizar o resultado (Status "SUCCESS" ou "FAILED").

```java
// JEITO ERRADO
@Transactional
public void process() {
    db.updateInitialStatus();
    api.callExternalGateway(); // <--- PERIGO: Conexão presa aqui!
    db.updateFinalStatus();
}

// JEITO CERTO
public void process() {
    Long id = service.savePending(); // @Transactional interno aqui
    
    var result = api.callExternalGateway(); // Fora de transação!
    
    service.updateResult(id, result); // Nova @Transactional aqui
}
```

## E se der erro no passo 3?

Se a API externa der OK mas você não conseguir atualizar o banco no passo 3, você terá uma inconsistência. Para resolver isso, você precisa de:
- **Idempotência:** A API externa deve permitir que você consulte o status ou tente novamente sem cobrar duas vezes.
- **Job de Reconciliação:** Um processo que roda em background procurando por registros que ficaram "PENDING" por muito tempo e verifica o status real na API externa.

## Checklist Anti-Sequestro de Conexões

Ao implementar integrações, valide se o seu código não está cometendo estes erros:
- [ ] Existe alguma chamada `restTemplate.exchange()` ou `feignClient` dentro de um método `@Transactional`?
- [ ] O timeout da API externa é menor que o timeout da transação do banco?
- [ ] O fluxo principal está dividido em: Iniciar (DB) -> Chamar (Rede) -> Finalizar (DB)?
- [ ] Existe um mecanismo de "retry" ou "fallback" para tratar falhas no passo final de atualização?
- [ ] O pool de conexões (HikariCP) está monitorado para detectar picos de uso durante latências de rede?