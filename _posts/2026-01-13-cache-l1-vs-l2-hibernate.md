---
title: "Cache de 1º vs 2º Nível no Hibernate"
date: 2026-01-13 09:00:00 -0300
categories: [Hibernate, JPA]
tags: [hibernate, jpa]
render_with_liquid: false
---

O Hibernate usa um sistema de cache em camadas para evitar idas desnecessárias ao banco de dados. Mas a diferença entre o Cache de Primeiro e Segundo nível confunde muitos desenvolvedores. Vamos descomplicar isso agora.

## Onde os Dados Vivem?

Imagine o Cache de 1º Nível como a sua **mochila** (pessoal e temporária) e o Cache de 2º Nível como o **armário do escritório** (compartilhado e persistente).

## 1. Cache de Primeiro Nível (L1)

- **Onde mora:** Na `Session` do Hibernate (ou `EntityManager`).
- **Escopo:** Transação atual. Cada thread/sessão tem o seu.
- **Obrigatoriedade:** Sempre ativo. Você não pode desabilitar.
- **Como funciona:** Se você buscar o `User 1` duas vezes na mesma transação, o Hibernate só vai ao banco na primeira vez. Na segunda, ele te entrega o que está na "mochila".
- **Vida útil:** Morre quando a `Session` é fechada.

## 2. Cache de Segundo Nível (L2)

- **Onde mora:** Na `SessionFactory` (escopo da aplicação).
- **Escopo:** Global. Compartilhado por todas as sessões e usuários.
- **Obrigatoriedade:** Opcional. Precisa ser configurado explicitamente.
- **Como funciona:** Se o `User A` buscar o `Produto 10`, ele vai para o L2. Quando o `User B` buscar o mesmo produto, ele o encontrará no cache global, mesmo estando em uma transação diferente.
- **Vida útil:** Vive enquanto a aplicação estiver rodando (ou conforme a política de expiração configurada).

## Tabela Comparativa

| Característica | Cache de 1º Nível (L1) | Cache de 2º Nível (L2) |
| :--- | :--- | :--- |
| **Escopo** | Sessão / Transação | Aplicação / Cluster |
| **Compartilhamento** | Privado da Thread | Compartilhado entre Threads |
| **Configuração** | Nativo (Automático) | Requer plugin (Ehcache, etc.) |
| **Dados** | Entidades monitoradas | Dados serializados (desconectados) |
| **Uso Principal** | Garantir identidade e Dirty Checking | Escalar performance em leituras frequentes |

## Quando usar cada um?

O **L1** você já usa, querendo ou não. Ele é fundamental para que o Hibernate saiba quais objetos estão sendo alterados e garantir que você não tenha duas instâncias diferentes representando a mesma linha do banco na mesma transação.

O **L2** deve ser usado com cautela. Ele é excelente para dados que são lidos com frequência mas mudam raramente (ex: categorias de produtos, configurações, estados/países).

## Conclusão: Da Mochila ao Almoxarifado

Para nunca mais esquecer: o **L1 Cache** é a sua **mochila** de trilha. Ela é leve, só você tem acesso e nela você guarda o que vai usar nos próximos quilômetros (na transação atual). Já o **L2 Cache** é o **almoxarifado** da base de apoio: ele é grande, compartilhado por todos os trilheiros (threads da aplicação) e guarda suprimentos que todos usam com frequência, mas buscar algo lá exige seguir protocolos de organização (configuração e plugins) para garantir que ninguém pegue mantimentos vencidos (dados obsoletos).