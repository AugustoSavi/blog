---
title: "Versionamento de APIs"
date: 2026-01-18 09:00:00 -0300
categories: [API Design]
tags: [api design]
render_with_liquid: false
---

Sua API é um sucesso e agora você precisa adicionar uma funcionalidade que quebra o contrato antigo. Como fazer isso sem derrubar os milhares de clientes que já usam sua API? A resposta é o **Versionamento**. Mas qual estratégia escolher?

## O Contrato Sagrado

Uma API é um contrato. Uma vez publicada, você não deve alterá-la de forma que quebre quem já a consome. O versionamento permite que a `v1` e a `v2` coexistam pacificamente.

## Estratégia 1: Versionamento na URL (Mais Comum)

Exemplo: `https://api.exemplo.com.br/v1/users`

**Prós:**
- Extremamente fácil de ver e testar no navegador.
- Cache de CDNs e Proxies funciona perfeitamente (URLs diferentes = recursos diferentes).
- Documentação clara (Swagger pode mostrar as duas versões facilmente).

**Contras:**
- Viola purismos do REST (o recurso "User" deveria ter apenas uma URI).
- Polui a URL.

## Estratégia 2: Versionamento por Header (Mais Elegante)

Exemplo: `GET /users` com o header `Accept: application/vnd.exemplo.v2+json` ou um custom header `X-API-Version: 2`.

**Prós:**
- Mantém as URLs limpas e semânticas.
- Segue o conceito de *Content Negotiation*.

**Contras:**
- Difícil de testar sem ferramentas como Postman/Curl.
- Problemas com cache (o cache precisa ser configurado para variar conforme o header `Vary: X-API-Version`).

## Política de Deprecation (Descontinuação)

Não basta criar a `v2`; você precisa avisar que a `v1` vai morrer.
- **Header Warning:** Envie um header avisando que aquela versão está obsoleta.
- **Sunset Header:** Informe a data exata em que a versão deixará de funcionar.
- **Comunicação:** E-mails, changelogs e documentação clara são vitais.

## Dica para evolução sem Quebra

Antes de criar uma nova versão, pergunte-se: "Posso fazer isso de forma retrocompatível?".
- Adicionar campos novos no JSON geralmente não quebra os clientes (se eles forem bem escritos).
- Mudar o nome de um campo ou o tipo de dado **sempre** quebra.

## Takeaway Prático: A Escolha Segura

Se você está começando um projeto novo e não quer perder tempo com configurações complexas de cache e negociação de conteúdo, **vá de versionamento na URL (`/v1/`)**. É a estratégia que oferece a melhor relação entre visibilidade, facilidade de depuração e compatibilidade com ferramentas de mercado. Reserve o versionamento por Header apenas para APIs internas ou cenários onde a estética da URL é um requisito de design inegociável.