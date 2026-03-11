---
title: "Azure Service Bus vs Kafka: Escolhendo a ferramenta de Mensageria certa"
date: 2026-02-24 09:00:00 -0300
categories: [Mensageria, Azure]
tags: [mensageria, azure, kafka]
render_with_liquid: false
---

Em ambientes que utilizam a nuvem da Microsoft, o **Azure Service Bus** é uma escolha comum para mensageria. Embora o Kafka seja o rei do streaming, o Service Bus brilha em cenários de mensageria empresarial clássica.

## Filas vs Logs

A principal diferença é o modelo mental:
- **Kafka:** É um Log de Eventos. As mensagens ficam lá mesmo após serem lidas. Ideal para alto volume e reprocessamento.
- **Azure Service Bus:** É um Broker de Mensagens tradicional. Foca em entrega confiável, filas e tópicos onde a mensagem costuma sumir após o processamento bem-sucedido.

## Recursos do Service Bus:

1.  **Dead Letter Queue (DLQ) Nativa:** Se uma mensagem falha 10 vezes, o Service Bus a move automaticamente para uma fila de erro para inspeção manual. No Kafka, você precisa implementar isso no seu código.
2.  **Scheduled Messages:** "Envie esta mensagem apenas daqui a 2 horas". Perfeito para fluxos de lembretes ou cobranças futuras.
3.  **Sessions:** Garante o processamento ordenado (FIFO) de mensagens relacionadas (ex: todas as mensagens do mesmo `orderId`) mesmo em consumidores paralelos.
4.  **Transactions:** Você pode processar uma mensagem e enviar outra de forma atômica. Ou as duas acontecem, ou a mensagem volta para a fila.

## Quando usar cada um?

- **Use Kafka** para o fluxo de transações de cartão, onde o volume é imenso e você precisa de auditoria e streaming de dados em tempo real.
- **Use Azure Service Bus** para fluxos de negócio complexos, como o gerenciamento de despesas, onde cada mensagem representa um comando que deve ser processado com garantias rigorosas e possíveis atrasos agendados.

## Conclusão

Não existe "melhor" ferramenta, apenas a ferramenta certa para o problema certo. Conhecer Azure Service Bus demonstra que você tem um repertório amplo de soluções para lidar com a complexidade de sistemas distribuídos e multi cloud.