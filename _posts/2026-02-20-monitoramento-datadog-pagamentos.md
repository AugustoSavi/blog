---
title: "Monitorando Pagamentos com Datadog: Além do \"Health Check\""
date: 2026-02-20 09:00:00 -0300
categories: [Observabilidade, DevOps]
tags: [observabilidade, devops, datadog]
render_with_liquid: false
---

Em um sistema financeiro, saber que o serviço está "UP" (200 OK) não é suficiente. Você precisa saber se as pessoas estão conseguindo gastar seus benefícios. O **Datadog** é a ferramenta de elite para essa observabilidade profunda.

## O Erro Silencioso

O serviço está rodando, a CPU está baixa, mas o número de transações aprovadas caiu 50% nos últimos 10 minutos. Nenhum alerta de infraestrutura disparou, mas o negócio está perdendo dinheiro. Isso é um erro de negócio que só a observabilidade real pega.

## 1. Métricas Customizadas (Business Metrics)

Não monitore apenas memória; monitore o **negócio**.
- **Métrica de Negócio (exemplo):** `transactions.approved.count` vs `transactions.denied.count`.
- Se a taxa de negação subir subitamente, algo está errado com o processador de cartões ou com a regra de saldo.

## 2. Dashboards de Funil

No Datadog, você pode criar um dashboard que mostra o caminho da transação:
`Requisição Recebida -> Validação de Saldo -> Chamada Gateway -> Resposta Usuário`.
Onde está o gargalo? Se o tempo da "Chamada Gateway" subiu, o problema é externo. Se a "Validação de Saldo" subiu, o banco de dados está lento.

## 3. Logs Estruturados e Facetas

Envie logs em JSON para o Datadog. Isso permite criar "Facetas" para filtrar instantaneamente:
- "Mostre todos os erros de pagamento para o `merchantId` X".
- "Quais usuários do estado de `SP` estão tendo timeout?".

## 4. APM e Distributed Tracing

Com o APM (Application Performance Monitoring) do Datadog, você vê o trace de uma transação cruzando 5 microserviços. Você descobre que o serviço de "Notificação" está atrasando a resposta final do pagamento porque está tentando enviar um SMS de forma síncrona.

## Dica: SLOs (Service Level Objectives)

Defina objetivos claros. "99.9% das transações devem ser processadas em menos de 200ms". O Datadog monitora esse erro de orçamento (*Error Budget*) e te avisa antes que você quebre o acordo de nível de serviço (SLA) com seus clientes.

## Conclusão

Observabilidade em sistemas críticos é sobre transparência e rapidez na resposta. Usar o Datadog para monitorar o sucesso das transações, e não apenas o status do servidor, é o que garante que os usuários nunca fiquem na mão na hora de usar seus benefícios.