---
title: "Design de APIs Públicas: Construindo Pontes para Parceiros Externos"
date: 2026-03-07 09:00:00 -0300
categories: [API, Arquitetura]
tags: [rest, oauth2, security, api-design, openapi]
render_with_liquid: false
---

Quando você constrói uma API interna, você tem o controle sobre quem a consome. Mas quando você decide abrir sua plataforma para parceiros externos, o jogo muda completamente. Sua API deixa de ser apenas um meio de transporte e se torna um **Produto**.

O design de uma API pública exige rigor em segurança, consistência de contrato e uma documentação impecável. Vamos ver como projetar uma API que os desenvolvedores externos amem usar.

## O Custo de um Contrato Quebrado

Diferente da sua API interna, onde você pode fazer um deploy coordenado com o Frontend, em uma API pública você não sabe quem são todos os seus clientes. Se você mudar o nome de um campo ou o tipo de dado, você **quebra o sistema do parceiro**.

---

## Estratégias de Design: O Guia de Sobrevivência

### 1. Autenticação e Autorização (O Portão)
Nunca use autenticação simples (Basic Auth). O padrão da indústria é o **OAuth 2.0**.
- **API Keys:** Úteis para identificação simples e rate limiting.
- **Client Credentials:** O parceiro (servidor) autentica-se com o seu servidor para obter um token JWT.

### 2. Idempotência (O Cinto de Segurança)
Como vimos em posts anteriores, toda operação que altera estado (ex: `POST /payouts`) deve aceitar uma `Idempotency-Key` no cabeçalho. Isso protege o parceiro contra falhas de rede e duplicação de pagamentos.

---

## Implementação: Rate Limiting e Quotas

Sua API pública deve ter limites claros para evitar que um parceiro mal escrito (ou mal-intencionado) derrube seu sistema.

```java
// Exemplo conceitual de Rate Limiting
@RateLimit(requests = 100, period = "1m")
@PostMapping("/v1/orders")
public ResponseEntity<OrderResponse> createOrder(...) {
    // ...
}
```

**As boas práticas sugerem:**
- Retornar o header `X-RateLimit-Remaining`.
- Em caso de excesso, retornar o status **429 Too Many Requests**.

---

## Funcionamento Interno: Versionamento na URL

Sempre coloque a versão na URL: `/v1/`, `/v2/`. 
Isso permite que você mantenha a `v1` funcionando por meses ou anos enquanto os parceiros migram gradualmente para a `v2`.

### O Uso de Webhooks

Frequentemente, uma API pública realiza processos demorados (ex: validação de KYC). Em vez de o parceiro ficar fazendo "polling" (perguntando a cada segundo), você deve implementar **Webhooks**. O parceiro fornece uma URL e o seu sistema envia um POST para ele quando o evento ocorrer.

```json
{
  "event": "kyc_approved",
  "partner_id": "P123",
  "data": { "userId": "U456", "status": "APPROVED" }
}
```

---

## Documentação: Swagger/OpenAPI e SDKs

Sua API é tão boa quanto sua documentação. Use o padrão **OpenAPI (Swagger)**.
- **Swagger UI:** Para exploração interativa.
- **SDK Generators:** Ofereça bibliotecas prontas em Java, Python ou Node para facilitar a vida do parceiro.

## Curiosidade Técnica: O Paradoxo da Retrocompatibilidade

Sabia que o Facebook e o GitHub mantêm versões antigas de suas APIs por anos? Adicionar campos novos ao JSON é seguro (quase sempre). Mas remover campos, mudar nomes ou alterar o comportamento de um status code (ex: mudar de 201 para 202) são **Breaking Changes**. Evite-as a todo custo dentro da mesma versão principal.

## Conclusão

Projetar APIs públicas é um exercício de disciplina técnica. Ao focar em **segurança, idempotência e versionamento**, você garante que sua plataforma seja escalável e confiável. Uma boa API pública é aquela que é invisível: o parceiro integra, o sistema funciona e ninguém precisa de suporte por meses.
