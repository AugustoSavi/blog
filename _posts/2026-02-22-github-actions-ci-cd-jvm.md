---
title: "GitHub Actions: Automatizando o Ciclo de Vida do seu App JVM"
date: 2026-02-22 09:00:00 -0300
categories: [DevOps, GitHub Actions]
tags: [devops, github actions, ci, cd]
render_with_liquid: false
---

O **GitHub Actions** tornou-se o padrão da indústria para automação de pipelines. Em um mundo onde o "Time-to-Market" é vital, ter um pipeline de CI/CD (Integração e Entrega Contínua) rápido e confiável é o que separa os times de elite do resto.

## Do Commit ao Deploy sem Stress

O GitHub Actions permite que você defina seu pipeline diretamente no repositório, usando arquivos YAML. Para uma aplicação Spring/Micronaut, o pipeline deve garantir que nenhum código "sujo" chegue à produção.

## Um Workflow Real para Java/Kotlin

```yaml
name: Java CI with Maven

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Set up JDK 21
      uses: actions/setup-java@v4
      with:
        java-version: '21'
        distribution: 'temurin'
        cache: maven
    
    - name: Build with Maven
      run: mvn clean install
    
    - name: Run Tests
      run: mvn test
```

## O que deve ser configurado no pipeline?

1.  **Cache de Dependências:** Não baixe a internet em todo commit. Use o `cache: maven` para acelerar o build de 5 minutos para 1 minuto.
2.  **Análise Estática:** Integre ferramentas como **SonarQube** ou **Checkstyle** para bloquear PRs que baixem a cobertura de testes ou introduzam vulnerabilidades.
3.  **Segurança (Secrets):** Nunca coloque chaves de API no YAML. Use os `GitHub Secrets` e injete-os como variáveis de ambiente.
4.  **Artifacts:** Após o build, gere a imagem Docker e envie para o seu Registry (ex: ECR da AWS) apenas se os testes passarem.

## Curiosidade: Matrix Builds

Você pode testar sua aplicação em múltiplas versões do Java (ex: 17 e 21) ou múltiplos Sistemas Operacionais simultaneamente usando a funcionalidade de `matrix` do GitHub Actions.

## O mantra da automação

Como diz a máxima do *Continuous Delivery*: "Se dói, faça mais vezes e automatize". O GitHub Actions é a ferramenta que permite transformar essa dor em um processo invisível e confiável. Ao investir tempo na automação do seu pipeline JVM, você não está apenas economizando minutos de build, mas protegendo a integridade do produto e a sanidade mental de quem o constrói.