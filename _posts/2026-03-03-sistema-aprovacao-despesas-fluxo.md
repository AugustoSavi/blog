---
title: "Sistemas de Aprovação de Despesas: Gerenciando Fluxos de Estado e Concorrência"
date: 2026-03-03 09:00:00 -0300
categories: [Arquitetura, Backend]
tags: [state-machine, backend, arquitetura, design-patterns, java]
render_with_liquid: false
---

Em qualquer sistema corporativo, despesas não são pagas instantaneamente. Existe uma jornada burocrática necessária: o funcionário cria a despesa, o gestor aprova (ou reprova), o financeiro libera e o banco paga. Se você implementar isso com um monte de `if (status == PENDING)`, seu código se tornará um labirinto impossível de manter.

A solução profissional para gerenciar fluxos complexos de negócio é o uso de **Máquinas de Estado (State Machines)**.

## A Explosão de Estados

À medida que o sistema cresce, as regras de transição mudam:
- "Não pode aprovar uma despesa que já foi paga."
- "Se a despesa for acima de R$ 5.000, precisa de uma segunda aprovação."
- "Uma despesa reprovada pode ser editada e reenviada?"

Codificar isso manualmente gera o que chamamos de **Anemic Domain Model**, onde a lógica de negócio está espalhada em Services gigantescos cheios de validações redundantes.

## A Solução: Máquina de Estados Finita (FSM)

Uma FSM define claramente:
1.  **Estados:** PENDING, APPROVED, REJECTED, PAID.
2.  **Eventos:** CREATE, APPROVE, REJECT, PAY, EDIT.
3.  **Transições:** Do PENDING para o APPROVED apenas através do evento APPROVE.

### Exemplo de Implementação com Design Pattern

```java
public enum ExpenseStatus {
    PENDING {
        @Override
        public ExpenseStatus approve() { return APPROVED; }
        
        @Override
        public ExpenseStatus reject() { return REJECTED; }
    },
    APPROVED {
        @Override
        public ExpenseStatus pay() { return PAID; }
    },
    REJECTED,
    PAID;

    public ExpenseStatus approve() { throw new IllegalStateTransitionException(); }
    public ExpenseStatus reject() { throw new IllegalStateTransitionException(); }
    public ExpenseStatus pay() { throw new IllegalStateTransitionException(); }
}
```

---

## O Desafio da Concorrência: Double Approval

Em sistemas distribuídos, dois gestores podem tentar clicar em "Aprovar" simultaneamente. Se não houver controle de concorrência, você pode processar a aprovação (ou pior, o pagamento) duas vezes.

**Estratégias de Proteção:**

### 1. Optimistic Locking (A melhor para a maioria dos casos)
Usa uma coluna de versão no banco de dados.

```java
@Entity
public class Expense {
    @Id private UUID id;
    private ExpenseStatus status;
    @Version private Long version; // O JPA cuida disso automaticamente
}
```
Se duas threads tentarem atualizar a mesma despesa, a segunda falhará com uma `OptimisticLockException`.

### 2. Pessimistic Locking (Para cenários críticos)
Bloqueia a linha no banco até o fim da transação.

```sql
SELECT * FROM expenses WHERE id = 'uuid' FOR UPDATE;
```

---

## Funcionamento Interno e Side-Effects

Uma transição de estado raramente muda apenas o status. Ela dispara **Side-Effects**: enviar notificações, atualizar o Ledger financeiro ou disparar um Job no Kafka.

Utilize o padrão **Domain Events**: o seu objeto `Expense` registra que uma transação ocorreu e o seu Framework (como o Spring Data DomainEvents) garante que o evento seja publicado apenas se a transação do banco for confirmada.

## Curiosidade Técnica: Spring Statemachine vs Workflow Engines

Se o seu fluxo for linear e simples, o padrão de design acima ou o **Spring Statemachine** resolvem. Se você tem fluxos humanos que levam dias (ex: "esperar o gestor aprovar por 48h antes de escalar"), considere **Workflow Engines** como **Temporal** ou **Camunda**. Eles gerenciam a persistência do estado do fluxo de forma resiliente e duradoura.

## Conclusão

Gerenciar despesas é gerenciar estados. Ao adotar Máquinas de Estado e controle rigoroso de concorrência, você transforma um processo caótico em um fluxo de dados previsível, auditável e seguro. Não deixe sua lógica de negócio virar um mar de `ifs`.
