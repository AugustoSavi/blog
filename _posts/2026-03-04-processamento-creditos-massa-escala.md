---
title: "Créditos em Massa: Como processar 100k transações simultâneas sem cair"
date: 2026-03-04 09:00:00 -0300
categories: [Arquitetura, Mensageria]
tags: [kafka, scalability, batch, java, microservices]
render_with_liquid: false
---

Em uma fintech de benefícios, o "Dia do Benefício" é o momento de maior pressão sobre a infraestrutura. Imagine uma empresa com 10.000 funcionários enviando uma planilha para creditar R$ 500 em cada cartão de uma só vez. Multiplique isso por milhares de empresas. O resultado? Uma avalanche de **escritas simultâneas** que destruiria qualquer banco de dados síncrono.

O segredo para sobreviver a esse pico não é ter um banco de dados maior, mas sim o uso de **Processamento em Lote (Batch) e Mensageria Assíncrona**.

## O Problema: O Gargalo da Transação Síncrona

Se você tentar processar o arquivo de crédito de forma síncrona na requisição HTTP:
1.  A conexão do usuário ficará aberta por minutos (gerando timeout).
2.  O pool de conexões do banco de dados será esgotado em segundos.
3.  Se um erro ocorrer no meio do arquivo de 10.000 linhas, como você retoma de onde parou sem duplicar créditos?

---

## A Solução: Arquitetura Orientada a Eventos (EDA)

O fluxo ideal para créditos em massa deve ser 100% assíncrono:

1.  **Ingestão:** O RH faz o upload do arquivo (S3) e a API salva um registro de "Processamento de Lote" (PENDING).
2.  **Resposta Rápida:** O servidor retorna `202 Accepted` imediatamente, com um ID para acompanhamento.
3.  **Particionamento de Eventos:** Um serviço (Producer) lê o arquivo e dispara mensagens individuais para o **Kafka**.

### O Design das Partições no Kafka

O Kafka escala através de partições. Se você quer processar 100k mensagens rápido, você precisa de múltiplas partições (ex: 50) e um **Grupo de Consumidores** proporcional. 

Cada consumidor processa uma fatia do lote de forma paralela.

---

## Implementação: Resiliência e Checkpoints

O maior desafio é garantir que nenhuma transação seja perdida. Para isso, usamos o conceito de **Idempotência no Consumidor**.

```java
@KafkaListener(topics = "credit-requests", groupId = "credit-processor")
public void consume(CreditRequestEvent event) {
    // 1. Usar o ID da transação original (Idempotency Key)
    String txKey = event.getTransactionId();

    // 2. Transação Atômica no Banco
    transactionTemplate.execute(status -> {
        // Se a transação já foi processada, o banco ignorará (UNIQUE constraint)
        ledgerService.credit(event.getAccountId(), event.getAmount(), txKey);
        
        // 3. Registrar o sucesso do processamento da linha
        batchRepository.markLineProcessed(event.getBatchId(), event.getLineNumber());
        
        return null;
    });
}
```

## Estratégias de Performance: Batch Updates

Processar uma linha por vez no banco de dados gera muitos "round-trips" de rede. Para otimizar, o seu consumidor pode agrupar mensagens e fazer um **Batch Insert**:

```sql
INSERT INTO ledger_entries (account_id, amount, tx_id) VALUES
(id1, 500, tx1), (id2, 500, tx2), ..., (id100, 500, tx100);
```
Isso reduz drasticamente a latência total do processamento.

## Funcionamento Interno: O "Consumer Lag"

Durante o dia do benefício, é normal que o Kafka tenha milhões de mensagens acumuladas (**Lag**). O importante é monitorar:
- **Throughput:** Quantas transações por segundo o cluster está processando?
- **Error Rate:** Qual a porcentagem de falhas (ex: conta inexistente)?
- **Dead Letter Queue (DLQ):** Mensagens que falharam devem ser enviadas para uma fila secundária para análise manual, sem travar a fila principal.

## Curiosidade Técnica: Backpressure

O Kafka tem uma característica genial: ele é baseado em **Pull**. O consumidor decide o ritmo de leitura. Se o banco de dados começar a ficar lento (CPU em 90%), o consumidor diminui a velocidade de processamento automaticCajuamente para não "afogar" o banco. Isso é o **Backpressure nativo**.

## Conclusão

Processar créditos em massa é a prova de fogo de qualquer arquiteto de software. Ao sair do modelo síncrono e adotar o Kafka com consumidores idempotentes e processamento batch, você transforma uma operação de risco em um processo escalável e resiliente. O usuário recebe o crédito rápido, e o banco de dados dorme tranquilo.
