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

Neste post, vamos analisar três abordagens para validação de CNPJs (incluindo cálculo de dígitos verificadores e suporte a caracteres alfanuméricos): o uso `String.matches`, `Pattern.compile` e a validação manual sem expressões regulares.

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

  private static boolean isValid(String cnpj) {
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

  private static final Pattern PADRAO_BASE = Pattern.compile("^[A-Z\\d]{12}$");
  private static final Pattern PADRAO_COMPLETO = Pattern.compile("^[A-Z\\d]{12}\\d{2}$");
  private static final Pattern PADRAO_ZERADO = Pattern.compile("^0+$");
  private static final Pattern PADRAO_FORMATACAO = Pattern.compile("[./-]");

  private static boolean isValid(String cnpj) {
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

  private static boolean isValid(String cnpj) {
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

## Comparação de Resultados

Abaixo, os resultados detalhados obtidos após processar um arquivo com **1.000.000 de registros** em cada modalidade em 10 execuções independentes.

| Estratégia | Execução | Tempo Total (s) | Tempo Java (ms) | Memória (RSS MB) |
| :--- | :---: | :---: | :---: | :---: |
| **matches com String** | #1 | 4.56s | 3308.36ms | 1175.49 MB |
| matches com String | #2 | 4.30s | 3205.98ms | 1170.77 MB |
| matches com String | #3 | 4.42s | 3306.97ms | 1431.42 MB |
| matches com String | #4 | 4.38s | 3273.83ms | 1156.36 MB |
| matches com String | #5 | 4.46s | 3255.76ms | 1234.29 MB |
| matches com String | #6 | 4.70s | 3482.45ms | 1247.73 MB |
| matches com String | #7 | 4.42s | 3152.46ms | 1129.54 MB |
| matches com String | #8 | 4.60s | 3360.58ms | 1164.21 MB |
| matches com String | #9 | 4.63s | 3460.75ms | 1280.86 MB |
| matches com String | #10 | 4.51s | 3305.28ms | 1239.43 MB |


| Estratégia | Execução | Tempo Total (s) | Tempo Java (ms) | Memória (RSS MB) |
| :--- | :--- | :--- | :--- | :--- |
| **Pattern.compile** | #1 | 2.84s | 1718.56ms | 449.47 MB |
| Pattern.compile | #2 | 2.65s | 1629.31ms | 429.09 MB |
| Pattern.compile | #3 | 2.49s | 1480.55ms | 454.18 MB |
| Pattern.compile | #4 | 2.64s | 1577.35ms | 430.80 MB |
| Pattern.compile | #5 | 2.68s | 1598.89ms | 466.20 MB |
| Pattern.compile | #6 | 2.73s | 1658.13ms | 450.72 MB |
| Pattern.compile | #7 | 2.66s | 1652.95ms | 451.61 MB |
| Pattern.compile | #8 | 2.57s | 1561.85ms | 441.00 MB |
| Pattern.compile | #9 | 2.57s | 1559.94ms | 436.63 MB |
| Pattern.compile | #10 | 2.64s | 1540.88ms | 429.25 MB |


| Estratégia | Execução | Tempo Total (s) | Tempo Java (ms) | Memória (RSS MB) |
| :--- | :--- | :--- | :--- | :--- |
| **Sem Regex** | #1 | 1.85s | 832.57ms | 364.95 MB |
| Sem Regex | #2 | 2.10s | 962.74ms | 343.99 MB |
| Sem Regex | #3 | 2.17s | 1037.10ms | 310.91 MB |
| Sem Regex | #4 | 2.05s | 981.12ms | 341.37 MB |
| Sem Regex | #5 | 1.93s | 901.66ms | 342.00 MB |
| Sem Regex | #6 | 1.87s | 833.01ms | 328.56 MB |
| Sem Regex | #7 | 1.88s | 807.12ms | 337.92 MB |
| Sem Regex | #8 | 2.22s | 1102.72ms | 321.79 MB |
| Sem Regex | #9 | 1.87s | 816.83ms | 334.28 MB |
| Sem Regex | #10 | 1.97s | 933.34ms | 327.39 MB |


### Tempo Interno

_Resultado obtido com `(end - start) / 1_000_000.0`_

Enquanto as tabela acima mostra o custo que o sistema operacional percebe, a tabela abaixo isola apenas o tempo gasto dentro do loop de processamento do Java, descartando o tempo de inicialização da JVM.

| Execução | Sem Compilar (ms) | Com Compilar (ms) | Sem Regex (ms) |
| :---: | :---: | :---: | :---: |
| #1 | 3308.36 ms | 1718.56 ms | 832.57 ms |
| #2 | 3205.98 ms | 1629.31 ms | 962.74 ms |
| #3 | 3306.97 ms | 1480.55 ms | 1037.10 ms |
| #4 | 3273.83 ms | 1577.35 ms | 981.12 ms |
| #5 | 3255.76 ms | 1598.89 ms | 901.66 ms |
| #6 | 3482.45 ms | 1658.13 ms | 833.01 ms |
| #7 | 3152.46 ms | 1652.95 ms | 807.12 ms |
| #8 | 3360.58 ms | 1561.85 ms | 1102.72 ms |
| #9 | 3460.75 ms | 1559.94 ms | 816.83 ms |
| #10 | 3305.28 ms | 1540.88 ms | 933.34 ms |

![Tempo de Execução](/assets/img/comparacao-tempo-regex.png){: .shadow .rounded-10 }
_Comparação do tempo total de execução em segundos._

![Uso de Memória](/assets/img/comparacao-memoria-regex.png){: .shadow .rounded-10 }
_Impacto no pico de consumo de memória (RSS)._

> **Disclaimer:** Estes resultados presentes nas imagens foram obtidos utilizando o comando `/usr/bin/time -v`. É importante notar que esses valores incluem o tempo total de carregamento da JVM, o carregamento de todas as classes, a leitura do arquivo em disco e a saída no console. Em sistemas de longa duração (como serviços web), a diferença real é ainda mais drástica, pois o overhead inicial da JVM é diluído ao longo do tempo.
{: .prompt-info }

### Ambiente de Teste

Para garantir a reprodutibilidade dos resultados, abaixo estão os detalhes do ambiente onde os benchmarks foram executados:

```text
OS: Ubuntu 24.04.3 LTS x86_64  
Kernel: 6.17.0-20-generic 
Shell: bash 5.2.21 
CPU: Intel i5-7200U (4) @ 2.500GHz 
GPU: Intel HD Graphics 620 
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

## Por que a Lógica Manual Vence?

Para uma validação simples como "verificar se os caracteres são alfanuméricos", o motor de regex é um exagero, é como usar uma bazuca para matar uma formiga.

Ao escrever `isAlphaNumeric(char c)`, você está dando uma instrução direta ao processador. Não há análise de grafos, não há decisões de backtracking. É pura comparação de bits, o que torna o processamento previsível e veloz.

## Dicas finais

1.  Se você vai usar uma regex mais de uma vez, **compile-a** em uma constante.
2.  **Validações Críticas:** Se o seu código está dentro de um loop que processa milhões de registros (ETLs, processamento de logs, gateways de pagamento), tente montar uma pipeline **sem o regex** pode te trazer muitos benefícios.
3.  **Legibilidade vs Performance:** Use regex para regras complexas onde o código manual se tornaria ilegível ou propenso a erros. O ganho de performance só justifica a complexidade extra se houver volume de dados suficiente.

