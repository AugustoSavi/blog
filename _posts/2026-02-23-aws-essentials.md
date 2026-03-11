---
title: "AWS Essentials: O que um Back-End precisa saber"
date: 2026-02-23 09:00:00 -0300
categories: [AWS, Cloud Computing]
tags: [aws, cloud computing]
render_with_liquid: false
---

Muitas empresas utilizam a **AWS (Amazon Web Services)** para sustentar seus microserviços. Para um desenvolvedor, a nuvem não é "apenas o computador de outra pessoa", mas um conjunto de ferramentas que mudam a forma como desenhamos software.

## Pensando em Cloud-Native

Não basta rodar o código; é preciso saber onde ele mora e como ele escala.

## Os "Top 5" Serviços AWS para Back-End:

1.  **Amazon EKS (Elastic Kubernetes Service):** É onde a maioria dos microserviços em larga escala rodam. Você deve entender como o EKS gerencia os pods e o escalonamento automático.
2.  **Amazon S3 (Simple Storage Service):** Usado para armazenar desde arquivos de notas fiscais até logs de auditoria. É virtualmente infinito e extremamente barato.
3.  **Amazon RDS (Relational Database Service):** Onde mora o MySQL. O foco aqui é entender réplicas de leitura, backups automáticos e Multi-AZ para alta disponibilidade.
4.  **AWS Lambda:** Para tarefas rápidas e eventos assíncronos (Serverless). Ideal para processar um webhook ou redimensionar uma imagem sem precisar de um servidor ligado 24/7.
5.  **Amazon DynamoDB:** O NoSQL chave-valor que já discutimos. Você deve dominar o conceito de *Capacity Units* e modelagem de tabelas.

## Segurança: O Pilar do IAM

O **IAM (Identity and Access Management)** é o serviço mais importante.
- **Regra de Ouro:** *Least Privilege* (Menor Privilégio). Seu microserviço só deve ter permissão para ler o bucket S3 que ele realmente usa, e nada mais.

## Curiosidade: Infraestrutura como Código (IaC)

Não crie recursos clicando no console da AWS. Use **Terraform** ou **AWS CDK**. Isso garante que o ambiente de Staging seja idêntico ao de Produção e que a infraestrutura possa ser destruída e recriada em minutos se necessário.

## Conclusão

Entender AWS é sair da "caixinha" do código e passar a ver o sistema como um todo. O conhecimento de nuvem permite que você projete soluções que não apenas funcionam, mas que são resilientes, seguras e economicamente viáveis.