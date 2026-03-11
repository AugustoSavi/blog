---
title: "OpenAPI (Swagger) Contract-First: A Verdade está no Contrato"
date: 2026-02-17 09:00:00 -0300
categories: [API Design, OpenAPI]
tags: [api design, openapi]
render_with_liquid: false
---

Muitos times usam o Swagger apenas como uma "página bonitinha" que documenta a API depois que ela está pronta. Mas, o **OpenAPI** deve ser o ponto de partida: o **Contract-First Development**.

## O Problema da Integração

O time de Frontend e o time de Mobile estão parados esperando o Back-end terminar a API. Quando o Back-end termina, o JSON não é o que o Front esperava. Esse vai-e-vem custa caro.

## O que é Contract-First?

Em vez de escrever o código e gerar o JSON do Swagger, você **escreve o arquivo YAML/JSON do OpenAPI primeiro**.

1.  **Acordo:** Back, Front e Product Owners discutem o contrato antes de qualquer linha de código.
2.  **Paralelismo:** Com o contrato definido, o Front pode criar um **Mock Server** (ex: Prism ou Stoplight) e começar a trabalhar imediatamente.
3.  **Geração de Código:** Use ferramentas como o `openapi-generator` para gerar as Interfaces e DTOs automaticamente tanto no Java/Kotlin quanto no TypeScript.

## Exemplo de Fluxo com Maven

Você coloca o arquivo `api.yaml` na pasta do projeto e o plugin do Maven gera as interfaces do Controller:

```xml
<plugin>
    <groupId>org.openapitools</groupId>
    <artifactId>openapi-generator-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals><goal>generate</goal></goals>
            <configuration>
                <inputSpec>${project.basedir}/src/main/resources/api.yaml</inputSpec>
                <generatorName>spring</generatorName>
                <configOptions>
                    <interfaceOnly>true</interfaceOnly>
                </configOptions>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## Vantagens

- **Consistência:** O código sempre reflete a documentação.
- **Segurança:** A validação de tipos e campos obrigatórios é feita automaticamente.
- **Menos Boilerplate:** Você não precisa escrever dezenas de DTOs manualmente; o gerador faz isso por você seguindo o padrão da indústria.

## Conclusão

Tratar sua API como um produto exige que o contrato seja levado a sério. O OpenAPI Contract-First elimina ambiguidades, acelera o desenvolvimento e garante que o seu sistema seja integrado de forma profissional e robusta.