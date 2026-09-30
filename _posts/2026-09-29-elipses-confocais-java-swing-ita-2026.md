---
title: "Elipses Confocais no Java: visualizando a questão 1 do ITA 2026"
date: 2026-09-29 10:00:00 -0300
categories: [Matematica, Java]
tags: [ita, geometria-analitica, elipse, swing, visualizacao]
description: "Transformando as relações entre excentricidade e área de duas elipses confocais em um experimento interativo com Java Swing."
math: true
mermaid: true
render_with_liquid: false
image:
  path: /assets/img/EllipseVisualizer.png
  alt: "Fluxo de decisão simplificado para escolha de abstração"
---


Eu estava vendo a resolução de **“UN Resolve - ITA 2026 - 1A FASE- Matemática”**, do canal [Universo Narrado Militares](https://www.youtube.com/@UniversoMilitares), quando a primeira questão chamou atenção por um motivo diferente da conta: ela descreve duas elipses que compartilham exatamente os mesmos focos. E eu gostaria de uma visualização dessa questão.

![Questão 1 ITA 2026 Matemática](/assets/img/questao-1-ita-2026-matematica.png){: .shadow .rounded-10 }

Em vez de guardar apenas que a resposta é $e_1 = \sqrt{10}/4$ = ~0,7905, decidi visualizar essa questão com java. O programa mantém os focos fixos, desenha as duas curvas e usa um slider para alterar $e_1$. A cada mudança, ele recalcula $e_2$, os semieixos e a razão entre as áreas. A solução deixa de ser um número isolado e passa a ser o ponto em que a tela mostra $A_2/A_1 = 6$.

## A informação que os focos compartilham

Para uma elipse de semieixo maior $a$, semieixo menor $b$ e distância focal $c$, valem as relações:

$$
e = \frac{c}{a}
$$

$$
b^2 = a^2 - c^2
$$

Na questão, $E_1$ e $E_2$ têm os mesmos focos. Portanto, $c$ tem o mesmo valor para as duas elipses. Isso é mais importante do que desenhar duas formas concêntricas: não basta terem o mesmo centro; os pontos focais precisam permanecer parados quando a excentricidade muda.

No código, escolhi $c = 1$. É uma unidade lógica, não um pixel. A escolha não altera a razão entre as áreas, mas facilita as fórmulas e dá ao programa uma referência fixa para posicionar $F_1$ e $F_2$.

Isolando os semieixos em função da excentricidade, temos:

$$
a = \frac{c}{e}
$$

$$
b = \frac{c}{e}\sqrt{1-e^2}
$$

Como a área de uma elipse é $A = \pi ab$, segue que:

$$
A = \pi\frac{c^2}{e^2}\sqrt{1-e^2}
$$

É essa expressão que o visualizador avalia para cada posição do slider.

## Por que o valor procurado é $\sqrt{10}/4$

O enunciado fornece duas relações:

$$
e_1 = 2e_2
\qquad\text{e}\qquad
A_2 = 6A_1
$$

Logo, $e_2=e_1/2$. Como $c$ é comum, ele e o fator $\pi$ se cancelam ao dividir as áreas:

$$
6 = \frac{A_2}{A_1}
  = 4\frac{\sqrt{1-e_1^2/4}}{\sqrt{1-e_1^2}}
$$

Elevando os dois lados ao quadrado depois de dividir por quatro:

$$
\frac{9}{4} = \frac{1-e_1^2/4}{1-e_1^2}
$$

$$
9(1-e_1^2) = 4-e_1^2
$$

$$
5 = 8e_1^2
\qquad\Longrightarrow\qquad
e_1 = \sqrt{\frac{5}{8}} = \frac{\sqrt{10}}{4} \approx 0{,}7906
$$

O slider inicia justamente nesse valor. Ao afastá-lo, a razão exibida no topo deixa de ser seis. Ao retorná-lo para aproximadamente $0{,}7906$, a relação reaparece sem que nenhum foco se mova.

{: .prompt-tip }
> Em uma elipse, diminuir $e$ com $c$ fixo aumenta $a$ e torna a curva menos alongada. Como $e_2=e_1/2$, $E_2$ é sempre a elipse externa e tem área maior que $E_1$.

## Do plano matemático aos pixels

O painel trabalha primeiro com unidades matemáticas. Para cada elipse, calcula `a` e `b`; somente na etapa de desenho multiplica esses valores por `scale`. Esse fator é escolhido a partir da maior elipse, para que ela caiba tanto na largura quanto na altura disponível do painel.

Há duas conversões relevantes:

- O centro matemático $(0, 0)$ é deslocado para o centro do painel em pixels.
- O eixo vertical da tela cresce para baixo, então o semieixo vertical é desenhado subtraindo pixels da coordenada Y do centro.

`Ellipse2D.Double` recebe a caixa delimitadora, e não o centro com os semieixos. Por isso a largura e a altura passadas ao construtor são $2a \cdot scale$ e $2b \cdot scale$.

No Swing, a renderização personalizada fica em `paintComponent`. A documentação de `JComponent` recomenda chamar a implementação da superclasse e usar uma cópia do contexto gráfico quando houver alterações no desenho; por isso o painel chama `super.paintComponent(graphics)` e cria `graphics.create()`.

## O visualizador completo

O arquivo abaixo usa apenas classes do JDK. O `JSlider` opera com inteiros de 50 a 990; dividir o valor por mil produz uma excentricidade no intervalo $[0{,}05, 0{,}99]$, evitando os limites degenerados $e=0$ e $e=1$.

```java
package codigos.ellipse;

import javax.swing.BorderFactory;
import javax.swing.JFrame;
import javax.swing.JLabel;
import javax.swing.JPanel;
import javax.swing.JSlider;
import javax.swing.SwingConstants;
import javax.swing.SwingUtilities;
import javax.swing.event.ChangeListener;
import java.awt.BasicStroke;
import java.awt.BorderLayout;
import java.awt.Color;
import java.awt.Dimension;
import java.awt.Font;
import java.awt.Graphics;
import java.awt.Graphics2D;
import java.awt.GridLayout;
import java.awt.RenderingHints;
import java.awt.geom.Ellipse2D;
import java.awt.geom.Line2D;
import java.text.DecimalFormat;
import java.text.DecimalFormatSymbols;
import java.util.Locale;

/**
 * Visualiza duas elipses confocais do problema e1 = 2e2 e A2 = 6A1.
 * O controle deslizante altera e1, mantendo os focos fixos e e2 = e1 / 2.
 */
public class EllipseVisualizer extends JPanel {

    private static final int WIDTH = 920;
    private static final int HEIGHT = 680;
    private static final double C = 1.0; // Distância fixa do centro a cada foco.
    private static final double SOLUTION_E1 = Math.sqrt(10.0) / 4.0;

    private static final Color BACKGROUND = new Color(16, 16, 16);
    private static final Color GRID = new Color(70, 90, 86);
    private static final Color ELLIPSE_1 = new Color(255, 184, 77);
    private static final Color ELLIPSE_2 = new Color(80, 255, 160);
    private static final Color FOREGROUND = new Color(225, 240, 232);
    private static final Color MUTED = new Color(165, 185, 175);

    private final DecimalFormat number = new DecimalFormat("0.0000",
            DecimalFormatSymbols.getInstance(Locale.of("pt", "BR")));
    private double e1 = SOLUTION_E1;

    public EllipseVisualizer() {
        setPreferredSize(new Dimension(WIDTH, HEIGHT));
        setBackground(BACKGROUND);
        setFont(new Font(Font.SANS_SERIF, Font.PLAIN, 14));
    }

    public void setE1(double e1) {
        this.e1 = Math.max(0.05, Math.min(0.99, e1));
        repaint();
    }

    public double getE1() {
        return e1;
    }

    private static double semiMajorAxis(double eccentricity) {
        return C / eccentricity;
    }

    private static double semiMinorAxis(double semiMajorAxis) {
        return Math.sqrt(semiMajorAxis * semiMajorAxis - C * C);
    }

    private static double area(double a, double b) {
        return Math.PI * a * b;
    }

    @Override
    protected void paintComponent(Graphics graphics) {
        super.paintComponent(graphics);
        Graphics2D g2 = (Graphics2D) graphics.create();
        g2.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);

        double e2 = e1 / 2.0;
        double a1 = semiMajorAxis(e1);
        double b1 = semiMinorAxis(a1);
        double a2 = semiMajorAxis(e2);
        double b2 = semiMinorAxis(a2);
        double area1 = area(a1, b1);
        double area2 = area(a2, b2);

        int centerX = getWidth() / 2;
        int centerY = getHeight() / 2 + 28;
        int margin = 74;
        double scale = Math.min((getWidth() - 2.0 * margin) / (2.0 * a2),
                (getHeight() - 172.0) / (2.0 * b2));

        drawTitle(g2, e2, area1, area2);
        drawAxes(g2, centerX, centerY, a1, b1, scale);
        drawEllipse(g2, centerX, centerY, a2, b2, scale, ELLIPSE_2, 3.2f);
        drawEllipse(g2, centerX, centerY, a1, b1, scale, ELLIPSE_1, 3.2f);
        drawFoci(g2, centerX, centerY, scale);
        drawLegend(g2, a1, b1, a2, b2, area1, area2);

        g2.dispose();
    }

    private void drawTitle(Graphics2D g2, double e2, double area1, double area2) {
        g2.setFont(getFont().deriveFont(Font.BOLD, 20f));
        g2.setColor(FOREGROUND);
        g2.drawString("Elipses confocais: e₁ = 2e₂", 28, 34);

        g2.setFont(getFont().deriveFont(15f));
        g2.setColor(MUTED);
        String result = "e₁ = " + number.format(e1)
                + "   |   e₂ = " + number.format(e2)
                + "   |   A₂ / A₁ = " + number.format(area2 / area1);
        g2.drawString(result, 28, 59);

        boolean atSolution = Math.abs(e1 - SOLUTION_E1) < 0.002;
        g2.setColor(atSolution ? ELLIPSE_2 : MUTED);
        String target = atSolution
                ? "Valor da solução: e₁ = √10 / 4  →  A₂ / A₁ = 6"
                : "Meta do problema: A₂ / A₁ = 6 quando e₁ = √10 / 4 ≈ "
                + number.format(SOLUTION_E1);
        g2.drawString(target, 28, 82);
    }

    private void drawAxes(Graphics2D g2, int x, int y, double a1, double b1, double scale) {
        int right = x + (int) Math.round(a1 * scale);
        int top = y - (int) Math.round(b1 * scale);

        g2.setStroke(new BasicStroke(1.3f));
        g2.setColor(GRID);
        g2.draw(new Line2D.Double(32, y, getWidth() - 32, y));
        g2.draw(new Line2D.Double(x, 106, x, getHeight() - 28));

        g2.setStroke(new BasicStroke(2.0f));
        g2.setColor(ELLIPSE_1.darker());
        g2.draw(new Line2D.Double(x, y, right, y));
        g2.draw(new Line2D.Double(x, y, x, top));
        drawMeasureLabel(g2, "a₁", (x + right) / 2, y - 9, ELLIPSE_1);
        drawMeasureLabel(g2, "b₁", x + 10, (y + top) / 2, ELLIPSE_1);

        g2.setColor(FOREGROUND);
        fillCircle(g2, x, y, 4);
        g2.setFont(getFont().deriveFont(13f));
        g2.drawString("centro", x + 8, y + 18);
    }

    private void drawEllipse(Graphics2D g2, int x, int y, double a, double b,
                             double scale, Color color, float strokeWidth) {
        double width = 2.0 * a * scale;
        double height = 2.0 * b * scale;
        g2.setColor(color);
        g2.setStroke(new BasicStroke(strokeWidth));
        g2.draw(new Ellipse2D.Double(x - width / 2.0, y - height / 2.0, width, height));
    }

    private void drawFoci(Graphics2D g2, int x, int y, double scale) {
        int leftFocus = x - (int) Math.round(C * scale);
        int rightFocus = x + (int) Math.round(C * scale);
        g2.setColor(new Color(255, 100, 100));
        fillCircle(g2, leftFocus, y, 7);
        fillCircle(g2, rightFocus, y, 7);
        g2.setFont(getFont().deriveFont(Font.BOLD, 14f));
        g2.drawString("F₁", leftFocus - 9, y - 13);
        g2.drawString("F₂", rightFocus - 9, y - 13);
    }

    private void drawLegend(Graphics2D g2, double a1, double b1, double a2, double b2,
                            double area1, double area2) {
        int x = 28;
        int y = getHeight() - 106;
        g2.setColor(new Color(20, 30, 27, 225));
        g2.fillRoundRect(x - 10, y - 24, 348, 101, 12, 12);
        g2.setColor(GRID);
        g2.drawRoundRect(x - 10, y - 24, 348, 101, 12, 12);

        g2.setFont(getFont().deriveFont(Font.BOLD, 14f));
        g2.setColor(ELLIPSE_1);
        g2.drawString("E₁", x, y);
        g2.setColor(FOREGROUND);
        g2.setFont(getFont().deriveFont(13f));
        g2.drawString("a₁ = " + number.format(a1) + "   b₁ = " + number.format(b1)
                + "   A₁ = " + number.format(area1), x + 28, y);

        g2.setFont(getFont().deriveFont(Font.BOLD, 14f));
        g2.setColor(ELLIPSE_2);
        g2.drawString("E₂", x, y + 25);
        g2.setColor(FOREGROUND);
        g2.setFont(getFont().deriveFont(13f));
        g2.drawString("a₂ = " + number.format(a2) + "   b₂ = " + number.format(b2)
                + "   A₂ = " + number.format(area2), x + 28, y + 25);

        g2.setColor(MUTED);
        g2.drawString("Focos fixos: c = 1   •   a = c/e   •   b² = a² − c²", x, y + 52);
    }

    private void drawMeasureLabel(Graphics2D g2, String text, int x, int y, Color color) {
        g2.setFont(getFont().deriveFont(Font.BOLD, 14f));
        g2.setColor(color);
        g2.drawString(text, x, y);
    }

    private static void fillCircle(Graphics2D g2, int x, int y, int radius) {
        g2.fillOval(x - radius, y - radius, radius * 2, radius * 2);
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            JFrame frame = new JFrame("Elipses Confocais");
            EllipseVisualizer visualizer = new EllipseVisualizer();

            int solutionValue = (int) Math.round(SOLUTION_E1 * 1000);
            JSlider eccentricitySlider = new JSlider(50, 990, solutionValue);
            eccentricitySlider.setMajorTickSpacing(100);
            eccentricitySlider.setMinorTickSpacing(25);
            eccentricitySlider.setPaintTicks(true);
            eccentricitySlider.setPaintLabels(true);

            JLabel valueLabel = new JLabel();
            valueLabel.setHorizontalAlignment(SwingConstants.CENTER);
            ChangeListener update = event -> {
                visualizer.setE1(eccentricitySlider.getValue() / 1000.0);
                double e1 = visualizer.getE1();
                valueLabel.setText("e₁ = " + visualizer.number.format(e1)
                        + "     e₂ = " + visualizer.number.format(e1 / 2.0));
            };
            eccentricitySlider.addChangeListener(update);
            update.stateChanged(null);

            JPanel controls = new JPanel(new GridLayout(2, 1, 0, 2));
            controls.setBorder(BorderFactory.createEmptyBorder(6, 16, 9, 16));
            controls.add(valueLabel);
            controls.add(eccentricitySlider);

            frame.setLayout(new BorderLayout());
            frame.add(visualizer, BorderLayout.CENTER);
            frame.add(controls, BorderLayout.SOUTH);
            frame.pack();
            frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
            frame.setLocationRelativeTo(null);
            frame.setVisible(true);
        });
    }
}

```

## O ciclo de interação do slider

O slider não move pontos diretamente. Ele representa $e_1$ em milésimos e dispara um `ChangeListener` para cada alteração. O listener atualiza o estado do painel com `setE1`, que limita o valor ao intervalo seguro e chama `repaint()`.

Na próxima pintura, `paintComponent` reexecuta toda a cadeia matemática: encontra $e_2$, calcula $a_1$, $b_1$, $a_2$, $b_2$, as duas áreas e a escala da tela. Como as coordenadas dos focos usam sempre a constante `C`, elas permanecem nos mesmos lugares relativos ao centro. O que muda são as caixas delimitadoras das elipses.

Esse desenho sob demanda é apropriado aqui: não há animação, nem thread de atualização contínua. O Swing agenda a repintura na sua fila de eventos, e a interface só redesenha quando o estado realmente muda.

## Compilando e explorando a solução

Na raiz do repositório, compile e/ou execute com:

```bash
java EllipseVisualizer.java
```

Ao abrir a janela, confira estes três passos:

- Comece em $e_1 \approx 0{,}7906$: o cabeçalho deve mostrar $A_2/A_1 \approx 6$.
- Arraste para valores menores: as duas elipses ficam menos excêntricas, mas os focos vermelhos permanecem fixos.
- Arraste para valores próximos de 1: $E_1$ se aproxima de uma forma mais alongada e a razão entre as áreas muda.

O valor exato não precisa coincidir com uma marca inteira do `JSlider`; o programa começa na posição inteira 791, isto é, $e_1=0{,}791$. A tolerância visual do cabeçalho foi definida para reconhecer a solução dentro de 0,002, enquanto o cálculo continua usando o valor real representado pelo slider.

## Uma conta que agora pode ser inspecionada

A parte mais útil deste pequeno programa não é substituir a resolução algébrica. É fornecer uma verificação geométrica para ela: com focos compartilhados, reduzir a excentricidade não é um ajuste arbitrário de largura; ele obriga a mudança simultânea dos dois semieixos e, por consequência, da área. Quando $e_1=\sqrt{10}/4$, a razão seis deixa de parecer uma condição abstrata do enunciado e passa a ser uma propriedade observável das duas curvas.


### Transparência

Este conteúdo foi gerado com auxílio de inteligência artificial (IA).
