---
title: "Mock, Stub ou Spy? Pare de confundir os termos nos seus testes!"
date: 2026-01-29 09:00:00 -0300
categories: [Testes, Mockito]
tags: [testes, mockito]
render_with_liquid: false
---

Você está escrevendo um teste unitário e precisa simular uma dependência. Você usa o Mockito e cria um... o quê? Um Mock? Um Stub? Muitas vezes usamos o termo "Mock" para tudo, mas na teoria dos testes (XUnit Patterns), eles têm papéis bem distintos.

## O Dublê de Teste (Test Double)

Assim como no cinema um dublê substitui o ator em cenas perigosas, no teste um **Test Double** substitui uma dependência real (como um banco ou uma API) que seria lenta ou difícil de configurar.

## 1. Stub (O Informante)

Um Stub fornece respostas prontas para as chamadas feitas durante o teste. Ele não se importa com quantas vezes foi chamado ou com quais parâmetros, ele apenas entrega o dado.
- **Foco:** Estado (Input para o seu código).
- **Exemplo:** "Sempre que perguntarem o saldo da conta 123, retorne 100.0".

## 2. Mock (O Investigador)

Um Mock é usado para verificar comportamentos. Ele registra quais métodos foram chamados e com quais argumentos, para que você possa validar se o seu código "fez a coisa certa".
- **Foco:** Interação (Behavior).
- **Exemplo:** "Verifique se o método `enviarEmail()` foi chamado exatamente uma vez com o endereço correto".

## 3. Spy (O Infiltrado)

Um Spy é um híbrido. Ele envolve um objeto **real** e permite que você monitore as chamadas a ele, mas mantendo o comportamento original (a menos que você decida mudar algo).
- **Foco:** Observação de objetos reais.
- **Exemplo:** "Use a classe de cálculo real, mas me diga quantas vezes o método `somar` foi invocado".

## Resumo Visual com Mockito

```java
// STUB: Fornecendo dados
when(repository.find(1L)).thenReturn(new User("João"));

// MOCK: Verificando comportamento
verify(repository, times(1)).save(any());

// SPY: Envolvendo objeto real
List<String> list = new ArrayList<>();
List<String> spyList = spy(list);
spyList.add("um");
verify(spyList).add("um"); // Verifica a interação
assertEquals(1, spyList.size()); // Comportamento real mantido
```

## Qual usar?

- Use **Stub** quando seu código precisa de um dado de uma dependência para continuar.
- Use **Mock** quando a ação principal do seu código é chamar um serviço externo (ex: enviar um SMS) e você quer garantir que isso aconteceu.
- Use **Spy** com cautela, geralmente em códigos legados onde você não consegue isolar totalmente as classes.

## Conclusão

Saber a diferença teórica ajuda a escrever testes mais claros e a se comunicar melhor com sua equipe. Lembre-se: **Stubs respondem, Mocks verificam.**