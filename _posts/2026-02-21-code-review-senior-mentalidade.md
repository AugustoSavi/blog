---
title: "Code Review: Como revisar código sem ser um chato (e focar no que importa)"
date: 2026-02-21 09:00:00 -0300
categories: [Cultura de Engenharia, Qualidade]
tags: [cultura de engenharia, qualidade]
render_with_liquid: false
---

Desenvolvedores participam ativamente de **Code Reviews**. Mas, revisar código não é apenas procurar por ponto e vírgula faltando ou erros de indentação (o Linter faz isso). É sobre garantir a saúde da arquitetura e mentorar o time.

## O Gargalo ou a Alavanca?

O Code Review pode ser o lugar onde o projeto trava ou onde o time aprende. A pessoa que esta revisando deve ser a alavanca que eleva a barra de qualidade sem criar atritos desnecessários.

---

## Além do Código

Um engenheiro não olha apenas para o `diff` do dia.

### 1. Roadmaps Técnicos
A evolução da arquitetura não deve parar a entrega de produto.  planejamentos de **Roadmaps Técnicos** que permitem mudanças estruturais (ex: migração de banco, quebra de serviço) de forma incremental. Cada entrega de funcionalidade deve ser um pequeno passo em direção à arquitetura desejada, evitando o "Big Bang Rewrite".

### 2. Mitigação de Riscos Arquiteturais
Identificar **Pontos Únicos de Falha (SPOF)** antes que eles ocorram. 
- "E se esse broker de mensagens cair?"
- "O que acontece se a latência dessa API externa triplicar?"
- "Temos um gargalo de escrita neste Shard específico?"
Mitigar riscos significa projetar redundância, timeouts e fallbacks como parte do design original, e não como um remendo após o incidente.

---

## 1. O que NÃO revisar (Deixe para a máquina)
- Indentação e espaços.
- Nomes de variáveis (se estiverem seguindo o padrão).
- Chaves em linhas erradas.
- **Dica:** Se você está comentando sobre estilo de código, seu time precisa de um **Checkstyle** ou **EditorConfig** no pipeline.

## 2. O que REALMENTE importa
- **Arquitetura:** O código está no lugar certo? Uma regra de negócio vazou para o Controller? Segue a Arquitetura Hexagonal?
- **Segurança:** Existe risco de SQL Injection? Dados sensíveis estão sendo logados? O usuário está autorizado a ver esse dado?
- **Performance:** Há um problema de N+1? Um loop está fazendo I/O desnecessário? O cache está sendo usado corretamente?
- **Simplicidade:** O código está "esperto demais"? Daqui a 6 meses, outro desenvolvedor conseguirá entender essa lógica complexa?

## 3. A Arte do Feedback Construtivo

Em vez de dizer "Isso está errado", tente:
- "O que você acha de usar o padrão Strategy aqui para facilitar a adição de novos tipos de benefícios no futuro?"
- "Notei que este loop pode ter problemas de performance com muitos dados. Já considerou usar paginação?"
- **Elogie também:** Se vir uma solução elegante, diga! O Code Review é uma ferramenta de incentivo.

## Curiosidade: O Ego no Code Review

Um desenvolvedor deve estar preparado para ter seu próprio código revisado por desenvolvedor Sênior, Pleno ou Junior. A humildade técnica é o que constrói uma cultura de engenharia forte.

## O que aprendi revisando código

Ao longo dos anos, percebi que os melhores revisores não são os que encontram mais bugs, mas os que fazem as melhores perguntas. Aprendi que um "Por que você escolheu este caminho?" abre muito mais espaço para o crescimento do time do que um "Isso deveria ser feito de outra forma". O Code Review é, acima de tudo, um exercício de empatia e comunicação técnica, onde o código é apenas o pretexto para construirmos engenheiros melhores.