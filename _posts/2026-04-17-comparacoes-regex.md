---
title: "Comparação String Match, Pattern.compile e Validação Manual"
date: 2026-04-21 00:00:01 -0300
categories: [Java, Performance]
tags: [regex, jvm, optimization, benchmark]
image:
  path: /assets/img/thumb-comparacao-regex.png
  alt: "Representação visual das operações realizadas"
mermaid: true
render_with_liquid: false
---

No dia a dia a conveniência muitas vezes atropela a performance. Um dos exemplos mais comuns é montar uma expressão no `String.matches()` e nunca mais mexer depois que fez funcionar. Essa abordagem pode se tornar um gargalo crítico em sistemas de alto rendimento, especialmente quando lidamos com grandes volumes de dados.

Neste post, vamos analisar quatro abordagens para validação de CNPJs (incluindo cálculo de dígitos verificadores e suporte a caracteres alfanuméricos): o uso `String.matches`, `Pattern.compile`, validação manual sem expressões regulares e uma versão otimizada da validação manual.

## Recompilação em toda execução

Quando utilizamos `String.matches(regex)`, a JVM não executa a busca imediatamente. Por baixo dos panos, o Java é obrigado a compilar a String da expressão regular em uma estrutura de dados complexa — uma máquina de estados finitos (NFA - Nondeterministic Finite Automaton) — antes de aplicá-la ao texto.

E ao usar esses métodos utilitários da classe `String`, a compilação ocorre **todas as vezes** que o método é invocado. Em um loop de processamento massivo, você está pagando o preço de "reaprender" a mesma regra repetidamente, gerando objetos temporários e desperdiçando ciclos de CPU.

## Fluxo de Processamento e Decisões

Para entender o ganho de performance, precisamos visualizar o que acontece quando dissociamos a compilação da execução ou quando removemos o motor de regex completamente em favor de uma lógica imperativa especializada.

![Comparaçõesfluxo](/assets/img/thumb-comparacao-regex.png){: .shadow .rounded-10 }
_Comparação dos fluxos das operações._

## Implementações em Detalhe

Para os testes, implementamos a validação completa de CNPJs, que agora permitem letras em sua composição. A lógica envolve a remoção de formatação, validação de estrutura e cálculo dos dois dígitos verificadores.

> Os algoritmos e regras de validação para o novo formato de CNPJ (alfanumérico) utilizados nos exemplos abaixo foram baseados na documentação técnica oficial disponibilizada pela Receita Federal [neste link](https://www.gov.br/receitafederal/pt-br/centrais-de-conteudo/publicacoes/documentos-tecnicos/cnpj/codigos-cnpj.zip/view).
{: .prompt-info }

### 1. Sem Compilar

Aqui, cada chamada a `matches` e `replaceAll` dispara uma nova compilação de regex internamente. É o padrão "confortável" que destrói a performance em escala.

```java
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.stream.Stream;

class SemCompilar {
  private static final int TAMANHO_CNPJ_SEM_DV = 12;
  private static final String REGEX_CARACTERES_FORMATACAO = "[./-]";
  private static final String REGEX_FORMACAO_BASE_CNPJ = "[A-Z\\d]{12}";
  private static final String REGEX_FORMACAO_DV = "[\\d]{2}";
  private static final String REGEX_VALOR_ZERADO = "^[0]+$";
  private static final int VALOR_BASE = (int) '0';
  private static final int[] PESOS_DV = { 6, 5, 4, 3, 2, 9, 8, 7, 6, 5, 4, 3, 2 };

  public static boolean isValid(String cnpj) {
    if (cnpj != null) {
      cnpj = removeCaracteresFormatacao(cnpj);
      if (isCnpjFormacaoValidaComDV(cnpj)) {
        String dvInformado = cnpj.substring(TAMANHO_CNPJ_SEM_DV);
        String dvCalculado = calculaDV(cnpj.substring(0, TAMANHO_CNPJ_SEM_DV));
        return dvCalculado.equals(dvInformado);
      }
    }
    return false;
  }

  private static String calculaDV(String baseCnpj) {
    if (baseCnpj != null) {
      baseCnpj = removeCaracteresFormatacao(baseCnpj);
      if (isCnpjFormacaoValidaSemDV(baseCnpj)) {
        String dv1 = String.format("%d", calculaDigito(baseCnpj));
        String dv2 = String.format("%d", calculaDigito(baseCnpj.concat(dv1)));
        return dv1.concat(dv2);
      }
    }
    throw new IllegalArgumentException(String.format("Cnpj %s não é válido para o cálculo do DV", baseCnpj));
  }

  private static int calculaDigito(String cnpj) {
    int soma = 0;
    for (int indice = cnpj.length() - 1; indice >= 0; indice--) {
      int valorCaracter = (int) cnpj.charAt(indice) - VALOR_BASE;
      soma += valorCaracter * PESOS_DV[PESOS_DV.length - cnpj.length() + indice];
    }
    return soma % 11 < 2 ? 0 : 11 - (soma % 11);
  }

  private static String removeCaracteresFormatacao(String cnpj) {
    return cnpj.trim().replaceAll(REGEX_CARACTERES_FORMATACAO, "");
  }

  private static boolean isCnpjFormacaoValidaSemDV(String cnpj) {
    return cnpj.matches(REGEX_FORMACAO_BASE_CNPJ) &&
        !cnpj.matches(REGEX_VALOR_ZERADO);
  }

  private static boolean isCnpjFormacaoValidaComDV(String cnpj) {
    return cnpj.matches(REGEX_FORMACAO_BASE_CNPJ.concat(REGEX_FORMACAO_DV)) &&
        !cnpj.matches(REGEX_VALOR_ZERADO);
  }

  void main(String[] args) throws IOException {
    var file = args.length > 0 ? args[0] : "data.txt";
    var entries = new HashMap<String, Boolean>();

    long start = System.nanoTime();

    try (Stream<String> stream = Files.lines(Paths.get(file))) {
      stream.forEach(item -> entries.put(item, isValid(item)));
    }

    long end = System.nanoTime();

    IO.println("Tempo de execução: " + (end - start) / 1_000_000.0 + " ms, itens mapeados: " + entries.size());
  }
}
```

### 2. Com Pattern.compile

Nesta versão, os padrões são compilados uma única vez e armazenados em constantes `static final`. O `Matcher` reutiliza a estrutura já processada.

```java
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.regex.Pattern;
import java.util.stream.Stream;

class ComCompilar {
  private static final int TAMANHO_CNPJ_SEM_DV = 12;
  private static final int VALOR_BASE = '0';

  private static final int[] PESOS_DV = {6, 5, 4, 3, 2, 9, 8, 7, 6, 5, 4, 3, 2};

  // Patterns pré-compilados
  private static final Pattern PADRAO_BASE = Pattern.compile("^[A-Z\\d]{12}$");
  private static final Pattern PADRAO_COMPLETO = Pattern.compile("^[A-Z\\d]{12}\\d{2}$");
  private static final Pattern PADRAO_ZERADO = Pattern.compile("^0+$");
  private static final Pattern PADRAO_FORMATACAO = Pattern.compile("[./-]");

  public static boolean isValid(String cnpj) {
    if (cnpj == null) return false;

    cnpj = removeCaracteresFormatacao(cnpj);

    if (!isCnpjFormacaoValidaComDV(cnpj)) return false;

    String base = cnpj.substring(0, TAMANHO_CNPJ_SEM_DV);
    String dvInformado = cnpj.substring(TAMANHO_CNPJ_SEM_DV);
    String dvCalculado = calculaDV(base);

    return dvCalculado.equals(dvInformado);
  }

  private static String calculaDV(String baseCnpj) {
    if (baseCnpj == null) {
      throw new IllegalArgumentException("CNPJ base nulo");
    }

    baseCnpj = removeCaracteresFormatacao(baseCnpj);

    if (!isCnpjFormacaoValidaSemDV(baseCnpj)) {
      throw new IllegalArgumentException("CNPJ inválido: " + baseCnpj);
    }

    int dv1 = calculaDigito(baseCnpj);
    int dv2 = calculaDigito(baseCnpj + dv1);

    return "" + dv1 + dv2;
  }

  private static int calculaDigito(String cnpj) {
    int soma = 0;
    int offset = PESOS_DV.length - cnpj.length();

    for (int i = 0; i < cnpj.length(); i++) {
      int valor = cnpj.charAt(i) - VALOR_BASE;
      soma += valor * PESOS_DV[offset + i];
    }

    int resto = soma % 11;
    return (resto < 2) ? 0 : 11 - resto;
  }

  private static String removeCaracteresFormatacao(String cnpj) {
    return PADRAO_FORMATACAO.matcher(cnpj).replaceAll("").trim();
  }

  private static boolean isCnpjFormacaoValidaSemDV(String cnpj) {
    return PADRAO_BASE.matcher(cnpj).matches()
      && !PADRAO_ZERADO.matcher(cnpj).matches();
  }

  private static boolean isCnpjFormacaoValidaComDV(String cnpj) {
    return PADRAO_COMPLETO.matcher(cnpj).matches()
      && !PADRAO_ZERADO.matcher(cnpj).matches();
  }

  public static void main(String[] args) throws IOException {
    var file = args.length > 0 ? args[0] : "data.txt";
    var entries = new HashMap<String, Boolean>();

    long start = System.nanoTime();

    try (Stream<String> stream = Files.lines(Paths.get(file))) {
      stream.forEach(item -> entries.put(item, isValid(item)));
    }

    long end = System.nanoTime();

    IO.println("Tempo de execução: " + (end - start) / 1_000_000.0 + " ms, itens mapeados: " + entries.size());
  }
}
```

### 3. Sem Regex

Aqui abandonamos completamente o motor genérico de expressões regulares. Utilizamos loops simples, `StringBuilder` e verificações de caracteres individuais.

```java
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.stream.Stream;

class SemRegex {
  private static final int TAMANHO_CNPJ_SEM_DV = 12;
  private static final int VALOR_BASE = '0';
  private static final int[] PESOS_DV = {6, 5, 4, 3, 2, 9, 8, 7, 6, 5, 4, 3, 2};

  public static boolean isValid(String cnpj) {
    if (cnpj == null) return false;

    cnpj = removeCaracteresFormatacao(cnpj);

    if (!isCnpjFormacaoValidaComDV(cnpj)) return false;

    String base = cnpj.substring(0, TAMANHO_CNPJ_SEM_DV);
    String dvInformado = cnpj.substring(TAMANHO_CNPJ_SEM_DV);
    String dvCalculado = calculaDV(base);

    return dvCalculado.equals(dvInformado);
  }

  private static String calculaDV(String baseCnpj) {
    if (baseCnpj == null) {
      throw new IllegalArgumentException("CNPJ base nulo");
    }

    baseCnpj = removeCaracteresFormatacao(baseCnpj);

    if (!isCnpjFormacaoValidaSemDV(baseCnpj)) {
      throw new IllegalArgumentException("CNPJ inválido para cálculo do DV: " + baseCnpj);
    }

    int dv1 = calculaDigito(baseCnpj);
    int dv2 = calculaDigito(baseCnpj + dv1);

    return "" + dv1 + dv2;
  }

  private static int calculaDigito(String cnpj) {
    int soma = 0;
    int offset = PESOS_DV.length - cnpj.length();

    for (int i = 0; i < cnpj.length(); i++) {
      int valor = cnpj.charAt(i) - VALOR_BASE;
      soma += valor * PESOS_DV[offset + i];
    }

    int resto = soma % 11;
    return (resto < 2) ? 0 : 11 - resto;
  }

  private static String removeCaracteresFormatacao(String cnpj) {
    StringBuilder sb = new StringBuilder(cnpj.length());

    for (int i = 0; i < cnpj.length(); i++) {
      char c = cnpj.charAt(i);
      if (c != '.' && c != '/' && c != '-') {
        sb.append(c);
      }
    }

    return sb.toString().trim();
  }

  private static boolean isCnpjFormacaoValidaSemDV(String cnpj) {
    if (cnpj.length() != TAMANHO_CNPJ_SEM_DV) return false;

    boolean allZero = true;

    for (int i = 0; i < cnpj.length(); i++) {
      char c = cnpj.charAt(i);

      if (!isAlphaNumeric(c)) return false;
      if (c != '0') allZero = false;
    }

    return !allZero;
  }

  private static boolean isCnpjFormacaoValidaComDV(String cnpj) {
    if (cnpj.length() != 14) return false;

    boolean allZero = true;

    for (int i = 0; i < 12; i++) {
      char c = cnpj.charAt(i);

      if (!isAlphaNumeric(c)) return false;
      if (c != '0') allZero = false;
    }

    // últimos 2 devem ser dígitos
    for (int i = 12; i < 14; i++) {
      char c = cnpj.charAt(i);

      if (!isDigit(c)) return false;
      if (c != '0') allZero = false;
    }

    return !allZero;
  }

  private static boolean isDigit(char c) {
    return c >= '0' && c <= '9';
  }

  private static boolean isAlphaNumeric(char c) {
    return (c >= '0' && c <= '9') ||
      (c >= 'A' && c <= 'Z');
  }

  void main(String[] args) throws IOException {
    var file = args.length > 0 ? args[0] : "data.txt";
    var entries = new HashMap<String, Boolean>();

    long start = System.nanoTime();

    try (Stream<String> stream = Files.lines(Paths.get(file))) {
      stream.forEach(item -> entries.put(item, isValid(item)));
    }

    long end = System.nanoTime();

    IO.println("Tempo de execução: " + (end - start) / 1_000_000.0 + " ms, itens mapeados: " + entries.size());
  }
}
```

### 4. Sem Regex (Otimizado)

Esta versão tenta levar a otimização manual ao extremo. Eliminamos a criação de objetos intermediários (como `String` e `StringBuilder`), utilizando arrays de caracteres fixos, desenrolamos loops (loop unrolling) para o cálculo dos DVs e fundimos a extração de caracteres com a validação básica.

```java
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.HashMap;
import java.util.stream.Stream;

public class SemRegexOtimizado {

    private static final int TAMANHO_CNPJ = 14;

    public static boolean isValid(String input) {
        if (input == null) return false;

        char[] c = new char[TAMANHO_CNPJ];
        int j = 0;

        // extração + validação + early exit
        for (int i = 0; i < input.length() && j < TAMANHO_CNPJ; i++) {
            char ch = input.charAt(i);

            if ((ch >= '0' && ch <= '9') || (ch >= 'A' && ch <= 'Z')) {
                c[j++] = ch;
            }
        }

        if (j != TAMANHO_CNPJ) return false;

        // todos zeros check inline (sem loop separado)
        boolean allZero = true;
        for (int i = 0; i < TAMANHO_CNPJ; i++) {
            if (c[i] != '0') {
                allZero = false;
                break;
            }
        }
        if (allZero) return false;

        // cálculo direto DV1 (loop unrolled)
        int s =
            (c[0]-'0') * 5 +
            (c[1]-'0') * 4 +
            (c[2]-'0') * 3 +
            (c[3]-'0') * 2 +
            (c[4]-'0') * 9 +
            (c[5]-'0') * 8 +
            (c[6]-'0') * 7 +
            (c[7]-'0') * 6 +
            (c[8]-'0') * 5 +
            (c[9]-'0') * 4 +
            (c[10]-'0') * 3 +
            (c[11]-'0') * 2;

        int r = s - 11 * (s / 11);
        int dv1 = (r < 2) ? 0 : 11 - r;

        // cálculo direto DV2
        s =
            (c[0]-'0') * 6 +
            (c[1]-'0') * 5 +
            (c[2]-'0') * 4 +
            (c[3]-'0') * 3 +
            (c[4]-'0') * 2 +
            (c[5]-'0') * 9 +
            (c[6]-'0') * 8 +
            (c[7]-'0') * 7 +
            (c[8]-'0') * 6 +
            (c[9]-'0') * 5 +
            (c[10]-'0') * 4 +
            (c[11]-'0') * 3 +
            dv1 * 2;

        r = s - 11 * (s / 11);
        int dv2 = (r < 2) ? 0 : 11 - r;

        return dv1 == (c[12] - '0') &&
               dv2 == (c[13] - '0');
    }

    void main(String[] args) throws IOException {
        var file = args.length > 0 ? args[0] : "data.txt";
        var entries = new HashMap<String, Boolean>();

        long start = System.nanoTime();

        try (Stream<String> stream = Files.lines(Paths.get(file))) {
          stream.forEach(item -> entries.put(item, isValid(item)));
        }

        long end = System.nanoTime();

        IO.println("Tempo de execução: " + (end - start) / 1_000_000.0 + " ms, itens mapeados: " + entries.size());
    }
}
```

## Comparação de Resultados

Abaixo, os resultados detalhados obtidos após processar um arquivo com **1.000.000 de registros** em cada modalidade em 100 execuções independentes.

![Tempo de Execução](/assets/img/comparacao-tempo-regex.png){: .shadow .rounded-10 }
_Comparação do tempo total de execução em segundos._

![Uso de Memória](/assets/img/comparacao-memoria-regex.png){: .shadow .rounded-10 }
_Impacto no pico de consumo de memória (RSS)._


Abaixo, os 10 primeiros de cada resultado obtido.

| Estratégia | Execução | Tempo Total (s) | Tempo Java (ms) | Memória (RSS MB) |
| :--- | :---: | :---: | :---: | :---: |
| **matches com String** | #1 | 5.74s | 4512.91ms | 1167.35 MB |
| matches com String | #2 | 5.97s | 4290.21ms | 1549.72 MB |
| matches com String | #3 | 4.45s | 3273.82ms | 1210.53 MB |
| matches com String | #4 | 4.59s | 3319.22ms | 1265.25 MB |
| matches com String | #5 | 4.63s | 3461.82ms | 1411.28 MB |
| matches com String | #6 | 4.77s | 3643.85ms | 1308.84 MB |
| matches com String | #7 | 4.53s | 3311.74ms | 1204.71 MB |
| matches com String | #8 | 4.54s | 3233.60ms | 1211.24 MB |
| matches com String | #9 | 4.45s | 3295.32ms | 1338.41 MB |
| matches com String | #10 | 4.55s | 3371.40ms | 1352.38 MB |


| Estratégia | Execução | Tempo Total (s) | Tempo Java (ms) | Memória (RSS MB) |
| :--- | :--- | :--- | :--- | :--- |
| **Pattern.compile** | #1 | 3.39s | 1748.88ms | 500.88 MB |
| Pattern.compile | #2 | 3.09s | 1905.52ms | 454.08 MB |
| Pattern.compile | #3 | 2.84s | 1669.36ms | 462.46 MB |
| Pattern.compile | #4 | 2.77s | 1554.83ms | 469.36 MB |
| Pattern.compile | #5 | 2.69s | 1587.82ms | 451.07 MB |
| Pattern.compile | #6 | 2.67s | 1606.12ms | 454.46 MB |
| Pattern.compile | #7 | 2.98s | 1878.03ms | 434.51 MB |
| Pattern.compile | #8 | 2.86s | 1597.64ms | 402.27 MB |
| Pattern.compile | #9 | 2.64s | 1588.42ms | 433.07 MB |
| Pattern.compile | #10 | 2.68s | 1600.39ms | 447.91 MB |


| Estratégia | Execução | Tempo Total (s) | Tempo Java (ms) | Memória (RSS MB) |
| :--- | :--- | :--- | :--- | :--- |
| **Sem Regex** | #1 | 2.47s | 926.41ms | 328.81 MB |
| Sem Regex | #2 | 2.68s | 1324.74ms | 336.83 MB |
| Sem Regex | #3 | 1.95s | 854.19ms | 335.47 MB |
| Sem Regex | #4 | 2.16s | 1001.36ms | 348.33 MB |
| Sem Regex | #5 | 2.30s | 1120.37ms | 312.73 MB |
| Sem Regex | #6 | 2.11s | 872.31ms | 369.33 MB |
| Sem Regex | #7 | 1.91s | 812.20ms | 311.07 MB |
| Sem Regex | #8 | 2.09s | 953.61ms | 382.05 MB |
| Sem Regex | #9 | 2.18s | 1081.28ms | 337.66 MB |
| Sem Regex | #10 | 2.27s | 954.62ms | 359.85 MB |


| Estratégia | Execução | Tempo Total (s) | Tempo Java (ms) | Memória (RSS MB) |
| :--- | :--- | :--- | :--- | :--- |
| **Sem Regex Otimizado** | #1 | 1.93s | 707.63ms | 280.72 MB |
| Sem Regex Otimizado | #2 | 2.34s | 532.44ms | 253.92 MB |
| Sem Regex Otimizado | #3 | 1.82s | 658.33ms | 319.54 MB |
| Sem Regex Otimizado | #4 | 1.75s | 510.32ms | 231.54 MB |
| Sem Regex Otimizado | #5 | 1.79s | 684.61ms | 322.10 MB |
| Sem Regex Otimizado | #6 | 1.66s | 570.10ms | 289.38 MB |
| Sem Regex Otimizado | #7 | 1.70s | 590.80ms | 323.61 MB |
| Sem Regex Otimizado | #8 | 1.81s | 580.85ms | 232.74 MB |
| Sem Regex Otimizado | #9 | 1.74s | 534.97ms | 268.29 MB |
| Sem Regex Otimizado | #10 | 1.64s | 540.97ms | 302.96 MB |


> **Disclaimer:** Estes resultados presentes nas imagens foram obtidos utilizando o comando `/usr/bin/time -v`. Esses valores incluem o tempo total de carregamento da JVM, o carregamento de todas as classes, a leitura do arquivo em disco e a saída no console. Em sistemas de longa duração (como apis rest), a diferença real pode ser ainda mais drástica, pois o overhead inicial da JVM é diluído ao longo do tempo.
{: .prompt-info }


## Tempos só do loop

_Resultado obtido com `(end - start) / 1_000_000.0` envolta do `try`_

![Tempo de Execução](/assets/img/comparacao-tempo-loop-regex.png){: .shadow .rounded-10 }
_Comparação do tempo total de execução em segundos só do loop._

Enquanto as tabela acima mostra o custo que o sistema operacional percebe, a tabela abaixo isola apenas o tempo gasto dentro do loop de processamento do Java, descartando o tempo de inicialização da JVM.


| Execução | Sem Compilar (ms) | Com Compilar (ms) | Sem Regex (ms) | Sem Regex Otimizado (ms) |
| :---: | :---: | :---: | :---: | :---: |
| #1 | 4512.91 ms | 1748.88 ms | 926.41 ms | 707.63 ms |
| #2 | 4290.21 ms | 1905.52 ms | 1324.74 ms | 532.44 ms |
| #3 | 3273.82 ms | 1669.36 ms | 854.19 ms | 658.33 ms |
| #4 | 3319.22 ms | 1554.83 ms | 1001.36 ms | 510.32 ms |
| #5 | 3461.82 ms | 1587.82 ms | 1120.37 ms | 684.61 ms |
| #6 | 3643.85 ms | 1606.12 ms | 872.31 ms | 570.10 ms |
| #7 | 3311.74 ms | 1878.03 ms | 812.20 ms | 590.80 ms |
| #8 | 3233.60 ms | 1597.64 ms | 953.61 ms | 580.85 ms |
| #9 | 3295.32 ms | 1588.42 ms | 1081.28 ms | 534.97 ms |
| #10 | 3371.40 ms | 1600.39 ms | 954.62 ms | 540.97 ms |


### Ambiente de Teste

Para garantir a reprodutibilidade dos resultados, abaixo estão os detalhes do ambiente onde os benchmarks foram executados:

```text
OS: Ubuntu 24.04.3 LTS x86_64  
Kernel: 6.17.0-20-generic 
Shell: bash 5.2.21 
CPU: Intel i5-7200U (4) @ 2.500GHz 
Memory: 19876MiB
```

**Versão do Java:**
```text
openjdk version "25.0.2" 2026-01-20
OpenJDK Runtime Environment (build 25.0.2+10-Ubuntu-124.04)
OpenJDK 64-Bit Server VM (build 25.0.2+10-Ubuntu-124.04, mixed mode, sharing)
```

### Análise dos Dados

- **Sem Compilar:** A abordagem mais custosa. O alto uso de memória reflete a criação e descarte frenético de objetos de regex e matchers. O tempo de execução sofre com o desperdício de ciclos em "re-analisar" a mesma regra um milhão de vezes.
- **Compilado:** Uma melhoria substancial. Ao pré-compilar os `Patterns`, eliminamos o overhead de análise sintática do regex em cada iteração. O consumo de memória estabiliza, mas ainda pagamos o preço de percorrer a NFA do motor de regex.
- **Sem Regex:** Ao implementar uma lógica manual, removemos toda a abstração do motor de regex. O Java consegue otimizar loops simples de forma extremamente eficiente via JIT compiler, resultando no menor tempo e menor pegada de memória.
- **Sem Regex Otimizado:** Ao removermos a utilização de `StringBuilder` e `substring`, e utilizarmos loop unrolling, permitimos que o JIT realize mais otimizações e o controle fino sobre a alocação de memória e o fluxo de execução faz toda a diferença.

## Por que a Lógica Manual Vence?

Para uma validação simples como "verificar se os caracteres são alfanuméricos", o motor de regex é um exagero, é como usar uma bazuca para matar uma formiga.

Ao escrever `isAlphaNumeric(char c)`, você está dando uma instrução direta ao processador. Não há análise de grafos, não há decisões de backtracking. É pura comparação de bits, o que torna o processamento previsível e veloz.

## Dicas finais

1.  Se você vai usar uma regex mais de uma vez, **compile-a** em uma constante.
2.  **Validações Críticas:** Se o seu código está dentro de um loop que processa milhões de registros (ETLs, processamento de logs, gateways de pagamento), tente montar uma pipeline **sem o regex** pode te trazer muitos benefícios.
3.  **Legibilidade vs Performance:** Use regex para regras complexas onde o código manual se tornaria ilegível ou propenso a erros. O ganho de performance só justifica a complexidade extra se houver volume de dados suficiente.

