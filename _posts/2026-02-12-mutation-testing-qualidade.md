---
title: "Mutation Testing: Por que 100% de cobertura pode ser uma mentira?"
date: 2026-02-12 09:00:00 -0300
categories: [Testes, Qualidade]
tags: [testes, qualidade]
render_with_liquid: false
---

Você olha para o relatório do Sonar e vê: "100% Code Coverage". Você sorri e faz o deploy. No dia seguinte, um bug bizarro aparece. Como isso é possível se todo o código foi testado? A resposta é: cobertura de código mede quais linhas foram executadas, mas não se os seus **assests** são bons. O **Mutation Testing** (Testes de Mutação) resolve isso.

## Quem testa os seus testes?

O Teste de Mutação funciona como um "vilão" que altera o seu código-fonte para ver se os seus testes percebem. Ele cria variantes do seu código (os **Mutantes**) com pequenas mudanças: troca um `>` por `>=`, um `+` por `-`, ou remove a chamada de um método.

## Como funciona o processo

1.  **Criação de Mutantes:** A ferramenta (ex: **PITest** para Java) altera uma linha de código.
2.  **Execução dos Testes:** Ela roda os seus testes contra esse código alterado.
3.  **Resultado:**
    - Se algum teste **falhar**, o mutante foi **morto** (Parabéns! Seu teste é bom).
    - Se todos os testes **passarem**, o mutante **sobreviveu** (Cuidado! Seu teste não está validando aquela lógica corretamente).

## Exemplo Prático

```java
// Código Original
public boolean isAdult(int age) {
    return age >= 18;
}

// Mutante criado pela ferramenta
public boolean isAdult(int age) {
    return age > 18; // Trocou >= por >
}
```

Se o seu teste só valida `isAdult(20)`, ele vai passar nos dois casos. O mutante sobreviveu! Isso indica que você esqueceu de testar o **valor de borda** (`isAdult(18)`).

## Por que usar?

1.  **Qualidade Real:** Dá uma métrica muito mais confiável que a cobertura de linha tradicional.
2.  **Encontra Testes Inúteis:** Revela testes que executam o código mas não validam nada (os famosos "testes para bater meta de coverage").
3.  **Educação:** Ajuda os desenvolvedores a entenderem melhor os limites e condições do seu próprio código.

## Curiosidade: O Custo Computacional

O maior defeito do Mutation Testing é ser lento. Como ele precisa rodar seus testes centenas de vezes para cada mutante, o tempo de execução pode ser alto. Por isso, costumamos rodar apenas nos componentes mais críticos do sistema.

## Checklist de Qualidade com Mutação

Antes de considerar sua tarefa concluída, passe pelo crivo dos mutantes:
- [ ] Meus testes cobrem os valores de borda (ex: `18` em um check de `>= 18`)?
- [ ] Se eu remover a chamada de um método secundário, algum teste falha?
- [ ] Se eu inverter um booleano de retorno, meus assertions capturam o erro?
- [ ] O score de mutação subiu junto com a cobertura de linhas?

Se a resposta for "sim" para todos, você não tem apenas cobertura; você tem confiança.