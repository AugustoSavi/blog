---
title: "Cursor-based vs Page-based"
date: 2026-01-22 09:00:00 -0300
categories: [API Design, Banco de Dados]
tags: [api design, banco de dados]
render_with_liquid: false
---

Se sua API retorna listas de dados, você precisa de paginação. Retornar 1 milhão de registros de uma vez vai derrubar seu servidor e travar o cliente. Mas você sabia que o tradicional `page=2&size=20` pode ser extremamente lento em tabelas grandes?

## O Problema do OFFSET

A paginação tradicional (**Page-based**) usa `LIMIT` e `OFFSET` no SQL. O problema é que, para te dar a página 1000, o banco de dados precisa ler as 20.000 linhas anteriores, descartá-las e só então te entregar as próximas 20. Isso gera um consumo de CPU e I/O que cresce conforme o usuário avança nas páginas.

## 1. Page-based Pagination (Offset)

Exemplo: `GET /orders?page=5&size=20`
SQL: `SELECT * FROM orders LIMIT 20 OFFSET 80`

**Vantagens:**
- Permite "pular" para qualquer página (ex: ir direto para a página 10).
- Fácil de implementar com Spring Data JPA (`Pageable`).
- O cliente sabe o total de páginas.

**Contras:**
- Performance ruim em datasets grandes.
- **Inconsistência de dados:** Se um item for deletado enquanto o usuário está navegando, ele pode ver o mesmo item duas vezes ou pular um item (Drift).

## 2. Cursor-based Pagination (Keyset)

Exemplo: `GET /orders?size=20&after_id=A123`
SQL: `SELECT * FROM orders WHERE id > 'A123' LIMIT 20`

**Vantagens:**
- **Performance constante:** O banco usa o índice para ir direto ao ponto. Não importa se você está no começo ou no fim da tabela.
- **Resiliente a mudanças:** Se itens forem inseridos ou deletados, o cursor continua apontando para o lugar certo.
- Ideal para "Infinite Scroll" (como o feed do Instagram ou Twitter).

**Contras:**
- Não permite pular para uma página específica (ex: "ir para a página 50").
- A ordenação deve ser feita por um campo único e sequencial (geralmente o ID ou Timestamp).

## Qual escolher?

- **Use Page-based** se o seu dataset é pequeno (milhares de registros) ou se o usuário realmente precisa saltar entre páginas (ex: um painel administrativo).
- **Use Cursor-based** para sistemas de alta escala, feeds de redes sociais, logs ou qualquer lista que cresça indefinidamente.

## Takeaway Prático: Offset ou Cursor?

Para decidir sua estratégia de paginação, avalie o volume de dados e o comportamento do usuário: se você está construindo um relatório administrativo onde o usuário precisa saltar para a página 50, o **OFFSET** é aceitável (desde que a tabela não tenha milhões de linhas). Se você está construindo uma API pública de alto tráfego ou um feed infinito, o **CURSOR** é obrigatório para garantir performance constante e evitar que a latência da sua query aumente à medida que o usuário "rola" a página.