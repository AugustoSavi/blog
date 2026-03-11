---
title: "Autenticação vs Autorização: Quem é você e o que você pode fazer?"
date: 2026-01-20 09:00:00 -0300
categories: [Segurança, API Gateway]
tags: [segurança, api gateway]
render_with_liquid: false
---

Muitos desenvolvedores usam os termos "Autenticação" e "Autorização" como se fossem a mesma coisa, mas no contexto de um API Gateway e Segurança de Sistemas, eles representam etapas completamente diferentes (e igualmente vitais).

## A Analogia do Hotel

Imagine que você chega em um hotel:
1.  **Autenticação:** Você apresenta seu RG na recepção para provar que você é o "João Silva". A recepção te entrega a chave do quarto 202.
2.  **Autorização:** Você tenta abrir a porta do quarto 305 com a sua chave. A porta não abre. Você está autenticado, mas não tem autorização para entrar naquele quarto específico.

## 1. Autenticação (AuthN) - "Quem é você?"

É o processo de validar a identidade do usuário ou sistema.
- **Como funciona:** O usuário fornece credenciais (usuário/senha, token JWT, biometria).
- **No API Gateway:** O Gateway verifica se o token é válido, se não expirou e se foi assinado por uma fonte confiável.
- **Resultado:** Um usuário autenticado (ou um erro 401 Unauthorized - ironicamente, o nome do status code HTTP 401 deveria ser "Unauthenticated").

## 2. Autorização (AuthZ) - "O que você pode fazer?"

É o processo de verificar se o usuário autenticado tem permissão para realizar uma ação específica.
- **Como funciona:** Verifica-se os papéis (Roles) ou permissões (Scopes) do usuário. "O João é um ADMIN?", "O João tem permissão de LEITURA?".
- **No API Gateway:** Pode ser feito via políticas (OPA - Open Policy Agent) ou apenas repassando o token para o microserviço decidir.
- **Resultado:** Acesso permitido (ou um erro 403 Forbidden).

## O Papel do API Gateway

Em uma arquitetura de microserviços, o API Gateway atua como a "primeira linha de defesa":

- **Offloading de AuthN:** O Gateway valida o JWT para que os microserviços não precisem repetir essa lógica. Se o token for inválido, a requisição nem chega ao microserviço, economizando recursos.
- **Global AuthZ:** O Gateway pode bloquear acessos a rotas sensíveis baseando-se apenas em scopes do token (ex: rotas `/admin/**` só para quem tem o scope `ROLE_ADMIN`).

## JWT: O Mensageiro

O token JWT é excelente porque ele pode carregar tanto a identidade (subject) quanto as permissões (claims/roles) em um único pacote assinado criptograficamente.

```json
{
  "sub": "joao_silva",
  "roles": ["USER", "PAYMENT_READ"],
  "exp": 1710000000
}
```

## Conclusão

Lembre-se: **Autenticação sempre vem antes da Autorização**. Você não pode saber o que alguém pode fazer antes de saber quem essa pessoa é. Em sistemas modernos, delegue a validação inicial para o Gateway, mas mantenha a lógica fina de autorização (regras de negócio) dentro dos seus serviços.