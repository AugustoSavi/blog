---
title: "Arquitetura de Notificações Omnichannel: Push, SMS e Email em Escala"
date: 2026-03-05 09:00:00 -0300
categories: [Arquitetura, Backend]
tags: [notifications, microservices, scalability, architecture, java]
render_with_liquid: false
---

Em sistemas modernos, notificações são o "pulso" da aplicação. O usuário quer saber o momento exato em que seu salário caiu, quando sua compra foi aprovada ou quando recebeu uma mensagem. Mas quando você tem milhões de usuários e múltiplos canais (Push, SMS, Email, WhatsApp), construir um serviço de notificação robusto e escalável é um desafio técnico fascinante.

O maior erro é acoplar a lógica de notificação diretamente no serviço de negócio. A solução é um **Notification Service** centralizado e desacoplado.

## Acoplamento e Latência

Imagine o `PaymentService` chamando a API do Twilio (SMS) ou do Firebase (Push) de forma síncrona:
1.  **Latência:** Se o Twilio demorar 5 segundos, o seu pagamento demora 5 segundos a mais.
2.  **Disponibilidade:** Se o Firebase estiver fora, o seu pagamento falha?
3.  **Complexidade:** O seu código financeiro não deveria saber o que é um `Device Token` do Android.

---

## A Solução: Arquitetura Orientada a Mensagens

O design ideal utiliza um padrão de **Fan-out** usando um Broker de mensagens (Kafka ou RabbitMQ).

1.  **Evento:** O `PaymentService` publica um evento `PaymentApprovedEvent` no Kafka.
2.  **Orquestração:** O `NotificationService` consome este evento e consulta as preferências do usuário (ex: ele quer Push? SMS?).
3.  **Entrega:** O sistema dispara jobs assíncronos para cada canal.

### O Modelo de Dados de Notificação

```java
public class NotificationRequest {
    private String userId;
    private NotificationTemplate template; // "PAYMENT_SUCCESS"
    private Map<String, String> parameters; // {amount: 50.00, shop: "Restaurante"}
    private List<Channel> preferredChannels; // [PUSH, SMS]
}
```

---

## Estratégias de Resiliência: O Desafio dos Provedores

Provedores externos (Twilio, SendGrid, Firebase) falham o tempo todo. Seu sistema deve ser resiliente:

### 1. Retries com Backoff Exponencial
Se o Firebase falhou, espere 1s, depois 2s, depois 4s... antes de tentar novamente.

### 2. Priorização de Filas (Queuing)
Diferencie notificações críticas de promocionais.
- **Fila Alta Prioridade:** OTP (One-Time Password) para login. Deve ser entregue em segundos.
- **Fila Baixa Prioridade:** "Você ganhou R$ 10 de cashback". Pode esperar minutos ou horas.

### 3. Rate Limiting por Usuário
Não queremos bombardear o usuário com 50 notificações seguidas. Implemente um **Throttling** para garantir que o usuário não receba mais de X notificações por hora em cada canal.

---

## Funcionamento Interno e Template Engine

Em vez de escrever strings no código, utilize uma **Template Engine** (como **Thymeleaf** ou **Handlebars**). 

```java
String body = templateEngine.process("payment_success_sms.txt", context);
smsClient.send(phoneNumber, body);
```
Isso permite que o time de marketing altere o texto das mensagens sem que você precise fazer um novo deploy do código.

## Curiosidade Técnica: O Id da Mensagem e o ID do Dispositivo

Muitas vezes, um usuário tem múltiplos dispositivos (um iPad e um Android). No Push, você envia para múltiplos **Device Tokens**, mas o usuário só quer ver uma notificação. O Notification Service deve gerenciar a lista de tokens ativos de cada `userId`, limpando tokens inválidos automaticamente sempre que o provedor retornar um erro de `ExpiredToken`.

## Conclusão

Arquitetura de notificações é sobre **desacoplamento** e **entrega garantida**. Ao isolar a complexidade dos canais e provedores em um serviço dedicado, você libera seu core financeiro para focar no que realmente importa, garantindo que a mensagem certa chegue ao usuário certo, no canal certo, no momento certo.
