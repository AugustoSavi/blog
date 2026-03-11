---
title: "Graceful Shutdown no Kubernetes"
date: 2026-01-31 09:00:00 -0300
categories: [DevOps, Kubernetes]
tags: [devops, kubernetes, spring]
render_with_liquid: false
---

Em um ambiente de microserviços, o deploy é um evento constante. No Kubernetes, quando um novo Pod é criado, o antigo deve ser removido. Se o seu serviço de pagamentos está processando uma transação importante e o Kubernetes simplesmente "mata" o processo (envia um `SIGKILL`), você tem uma transação interrompida e um cliente insatisfeito.

O **Graceful Shutdown** (Desligamento Suave) é o processo onde a sua aplicação recebe um sinal de encerramento (`SIGTERM`), para de aceitar novas requisições, mas termina de processar o que já está em andamento antes de fechar.

## Configuração no Spring Boot

A partir do Spring Boot 2.3, habilitar o graceful shutdown é simples:

```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

Com isso, o Spring esperará até 30 segundos para que as requisições ativas terminem.

## Kubernetes

Mesmo com o Spring configurado, o Kubernetes pode ser mais rápido que a propagação da rede. Ele remove o Pod do `Service` e envia o `SIGTERM` quase simultaneamente. Isso pode causar erros 504 (Gateway Timeout) se o Ingress tentar enviar tráfego para um Pod que já está "morrendo".

A solução é usar um `preStop` hook no seu `deployment.yaml`:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 5"]
```

## Por que 5 segundos de 'sleep'?

O comando `sleep 5` garante que o seu app continue "vivo" e aceitando requisições por mais 5 segundos enquanto o Kubernetes atualiza os Endpoints e remove o Pod do serviço de roteamento. Só depois desses 5 segundos é que o sinal `SIGTERM` é enviado para o Spring.

## Checklist para Deploys Sem Erros

Garanta que sua aplicação sobrevive ao ciclo de vida do Kubernetes validando estes pontos:
- [ ] O Spring Boot está com `server.shutdown=graceful` ativo?
- [ ] O `timeout-per-shutdown-phase` é compatível com o `terminationGracePeriodSeconds` do seu Pod? (O K8s deve esperar mais que o Spring).
- [ ] Existe um `preStop` hook com `sleep` para dar tempo de a rede (Ingress/Service) se atualizar?
- [ ] Sua aplicação trata corretamente o sinal `SIGTERM` para fechar conexões de banco e brokers?
- [ ] Os logs confirmam que o processo terminou com sucesso (`exit 0`) após o sinal de desligamento?