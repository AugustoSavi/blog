---
title: "Evolução de Banco de Dados: Flyway e o Zero Downtime Deploy"
date: 2026-02-19 09:00:00 -0300
categories: [DevOps, Banco de Dados]
tags: [devops, banco de dados, flyway]
render_with_liquid: false
---

Você mudou o nome de uma coluna no seu código local, mas esqueceu de avisar o time de DevOps. O deploy acontece, o app tenta ler a coluna nova, ela não existe e... BUM! O sistema cai. Gerenciar migrações de banco de dados manualmente é um erro fatal. Conheça o **Flyway**.

## Versionamento de Banco

Trate o seu banco de dados como o seu código. Ele precisa de histórico, versões e automação. O Flyway (ou Liquibase) faz exatamente isso.

## Como o Flyway funciona?

Ele guarda uma tabela chamada `flyway_schema_history` no seu banco. Sempre que sua aplicação sobe, o Flyway olha a pasta de scripts (`src/main/resources/db/migration`), vê quais scripts ainda não foram executados e os aplica em ordem.

**Exemplo de Arquivo:** `V1__create_accounts_table.sql`
**Exemplo de Arquivo:** `V2__add_status_to_accounts.sql`

## O Desafio: Zero Downtime Migrations

Em sistemas de alta disponibilidade, você não pode desligar o banco para fazer uma migração. Algumas migrações são perigosas (como o `ALTER TABLE` em tabelas com milhões de linhas).

**Regras de Ouro:**
1.  **Nunca delete ou renomeie colunas de imediato:** Use o padrão *Expand and Contract*. Primeiro adicione a nova, depois migre os dados, e só no próximo deploy remova a antiga.
2.  **Evite Default Values pesados:** Adicionar uma coluna com `DEFAULT 'ACTIVE'` em uma tabela de 100 milhões de linhas pode travar o banco por minutos.
3.  **Cuidado com Locks:** Algumas operações bloqueiam a tabela para escritas. Verifique se o seu banco (MySQL 8, por exemplo) suporta *Instant DDL* ou use ferramentas como o **gh-ost** ou **pt-online-schema-change**.

## Flyway no Spring Boot / Micronaut

Basta adicionar a dependência no `pom.xml` e colocar os scripts na pasta certa. O framework se encarrega de rodar a migração no momento do startup da aplicação.

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

## Conclusão

Engenharia de software moderna exige automação. O Flyway garante que o seu banco de dados evolua junto com o seu código, de forma segura, auditável e sem surpresas no momento do deploy.