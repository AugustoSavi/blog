---
title: "Richardson Maturity Model: Sua API é realmente REST?"
date: 2026-02-09 09:00:00 -0300
categories: [API Design, REST]
tags: [api design, rest]
render_with_liquid: false
---

Você criou um endpoint que recebe um JSON e retorna um 200 OK. Você diz que sua API é RESTful. Mas, de acordo com Leonard Richardson, existem quatro níveis de maturidade para uma API atingir o "Glória do REST". Em qual nível você está?

## A Escada para a Glória

O Modelo de Maturidade de Richardson define passos claros para transformar um serviço web comum em um sistema verdadeiramente hiperfocado em recursos.

## Nível 0: O Pântano do POX (Plain Old XML/JSON)
Você usa o HTTP apenas como um túnel para transportar dados. Geralmente um único endpoint (`/api/service`) que recebe ordens via POST.
- **Exemplo:** SOAP ou APIs que usam `POST /getUsers` e `POST /deleteUser`.

## Nível 1: Recursos
Você começa a usar URLs para identificar "coisas" (Recursos).
- **Exemplo:** `/users`, `/orders/123`. Mas você ainda usa o mesmo verbo (geralmente POST) para tudo.

## Nível 2: Verbos HTTP
Aqui é onde a maioria das boas APIs corporativas moram. Você usa os verbos HTTP para o que eles foram criados.
- **GET** para buscar, **POST** para criar, **PUT** para atualizar, **DELETE** para remover.
- Você também usa os **Status Codes** corretos (201 Created, 404 Not Found, 409 Conflict).

## Nível 3: HATEOAS (Hypermedia As The Engine Of Application State)
O nível final. Sua API não retorna apenas dados, ela retorna **links** para as próximas ações possíveis.
- **Exemplo:** Ao buscar um pedido, a resposta inclui links para "cancelar", "pagar" ou "acompanhar entrega". O cliente não precisa "adivinhar" as URLs.

## Exemplo Nível 3 (JSON):
```json
{
  "orderId": 55,
  "status": "AWAITING_PAYMENT",
  "links": [
    { "rel": "pay", "href": "/orders/55/pay" },
    { "rel": "cancel", "href": "/orders/55/cancel" }
  ]
}
```

## Conclusão

Mirar no Nível 2 é o requisito básico. O Nível 3 (HATEOAS) é o ideal acadêmico, mas muitas vezes evitado pela complexidade de implementação no lado do cliente. O importante é sair do Nível 0 e dar semântica real às suas URIs e verbos.