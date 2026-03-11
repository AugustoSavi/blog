---
title: "Framework de 7 Passos: Como Engenheiros de FAANG abordam System Design"
date: 2026-03-01 09:00:00 -0300
categories: [Arquitetura]
tags: [system-design, arquitetura]
render_with_liquid: false
---

Você já esteve em uma entrevista de arquitetura onde o entrevistador soltou a fatídica pergunta: "Como você projetaria o WhatsApp?". Se o seu primeiro impulso foi abrir o quadro branco e desenhar uma caixa escrita "Servidor" e outra escrita "Banco de Dados", você provavelmente falhou.

Engenheiros de boas empresas não começam pelo desenho. Eles começam pelo **entendimento**. Para isso, utilizam um framework mental que garante que nenhuma decisão de design seja tomada sem justificativa técnica.

## Por que um Framework?

Em sistemas complexos, as possibilidades são infinitas. O objetivo do entrevistador não é ver um desenho perfeito, mas sim observar o seu **processo de tomada de decisão**. O framework de 7 passos serve para guiar esse raciocínio e evitar o "paralisia por análise".

---

## Passo 1: Clarificar o Problema

Antes de propor qualquer solução, você deve entender as restrições. Pergunte sobre:
- **Usuários:** Quantos usuários ativos (DAU)?
- **Tráfego:** Qual o QPS (Queries Per Second) médio e de pico?
- **Dados:** Qual o volume total de dados esperado para os próximos 5 anos?
- **Consistência:** O sistema aceita consistência eventual ou exige consistência forte (ACID)?

> "Esse sistema de pagamentos precisa suportar liquidação em tempo real ou pode ser processado em lotes?" – Esta pergunta vale mais que um desenho de microserviço.
{: .prompt-tip }

---

## Passo 2: Requisitos Funcionais

Defina o escopo mínimo viável (MVP) do que o sistema **deve fazer**. 
Exemplo para um sistema de benefícios:
- Criar carteira.
- Adicionar crédito.
- Realizar pagamento.
- Consultar histórico.

---

## Passo 3: Requisitos Não-Funcionais

Aqui é onde você demonstra entendimento. Defina as metas de qualidade:
- **Latência:** < 100ms para consultas de saldo.
- **Disponibilidade:** 99.99% (Four nines).
- **Escalabilidade:** Suportar picos de 10x o tráfego médio.
- **Segurança:** Criptografia em repouso e em trânsito.

---

## Passo 4: Capacity Planning (Estimativa de Escala)

Engenheiros fazem contas. Estimar a carga ajuda a escolher o banco de dados e a estratégia de cache.

**Exemplo rápido:**
- 10 milhões de usuários.
- 10 transações por dia por usuário.
- total: 100 milhões de transações/dia ≈ 1.150 transações/segundo (TPS).
- Pico (10x): 11.500 TPS.

---

## Passo 5: Arquitetura de Alto Nível

Agora sim, desenhe os grandes blocos. Foque no fluxo principal dos dados:
1.  **Clients** (Mobile/Web).
2.  **API Gateway** (Auth, Rate Limit).
3.  **Services** (Business Logic).
4.  **Database/Cache** (Persistência).

---

## Passo 6: Aprofundar Componentes Críticos

Escolha 1 ou 2 partes vitais e detalhe. Se for um sistema financeiro, fale sobre **Idempotência** e **Locks Distribuídos**. Se for um sistema de chat, fale sobre **WebSockets** e **Pub/Sub**.

---

## Passo 7: Trade-offs e Conclusão

Nenhuma arquitetura é perfeita. Toda escolha tem um custo.
- "Escolhi MySQL por causa das transações ACID, embora o Cassandra escalasse melhor para escrita, pois a consistência financeira é inegociável aqui."

## Curiosidade Técnica: A Falácia da Disponibilidade

Muitos candidatos dizem "meu sistema é 100% disponível". **100% não existe.** Até o Google Cloud e a AWS têm SLAs de 99.9% a 99.99%. Dizer 100% mostra falta de experiência com sistemas reais que falham (e falham o tempo todo).

## Takeaway prático

Para sua próxima discussão de arquitetura, imprima mentalmente estes 7 passos: **Clarificar -> Funcionais -> Não-Funcionais -> Escala -> Alto Nível -> Detalhamento -> Trade-offs.** Começar pelo Passo 1 (Clarificação) em vez de pular direto para o Passo 5 (Desenho) é o que demonstra senioridade e evita que você projete um canhão laser para matar uma mosca (ou um sistema frágil para um tráfego massivo).
