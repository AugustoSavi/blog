---
title: "Projeção 3D com Java Swing: Renderizando um Cubo do Zero"
date: 2026-04-09 10:00:00 -0300
categories: [Java, ComputacaoGrafica]
tags: [swing, 3d, math]
image:
  path: /assets/img/cubo.png
  alt: "Interface do Visualizador de Cubo 3D em Java Swing"
math: true
render_with_liquid: false
---

Sempre tive dificuldade com a matemática puramente teórica. O que mais me fazia falta nas aulas era a conexão entre o conceito abstrato e a aplicação prática. Calcular seno e cosseno parecia um exercício vazio, até mais tarde encontrar que essas fórmulas eram o segredo para fazer os itens coletáveis girar suavemente no centro da tela de alguns jogos.

Hoje, vamos resgatar essa lógica para construir um visualizador de cubo do zero usando apenas Java Swing, transformando fórmulas "chatas" em uma visualização interativa.

## 3D para 2D

Para entender como essa projeção acontece, precisamos primeiro aceitar que o desafio de renderizar 3D em uma tela plana é, essencialmente, um problema de mapeamento. Precisamos transformar coordenadas $(x, y, z)$ em coordenadas $(x, y)$ que façam sentido para nos. Para isso, estruturamos o que chamamos de pipeline de renderização.

O fluxo de renderização segue um "pipeline" clássico:
1. **Definição de Vértices:** Onde os pontos estão no espaço local.
2. **Rotação:** Girar os pontos no espaço nos eixos X e Y.
3. **Translação:** Mover o objeto (neste caso, empurrá-lo para "dentro" da tela no eixo Z).
4. **Projeção:** Aplicar a perspectiva.
5. **Mapeamento de Tela:** Converter unidades matemáticas para pixels.

![Visualizador de Cubo 3D](/assets/img/cubo.png){: .shadow .rounded-10 w="600" h="400" }
_Interface do visualizador com controles de rotação e distância._

## A Trigonometria da Rotação: Seno e Cosseno

Para que o cubo "vire" na tela, precisamos de funções que descrevam movimentos circulares. É aqui que entram o **Seno (sin)** e o **Cosseno (cos)**. 

Imagine um ponto em um círculo unitário. O cosseno de um ângulo $\theta$ representa a posição horizontal ($x$) desse ponto, enquanto o seno representa a posição vertical ($y$). Quando aplicamos isso a um objeto 3D, estamos essencialmente rotacionando seus vértices em torno de um eixo fixo.

Para rotacionar um ponto $(x, y)$ em um ângulo $\theta$ no plano 2D (que é a base para as rotações 3D nos eixos X, Y ou Z), utilizamos a seguinte transformação linear:

$$
x' = x \cos(\theta) - y \sin(\theta)
$$

$$
y' = x \sin(\theta) + y \cos(\theta)
$$

No nosso código Java, aplicamos essa lógica para "girar" o cubo. Por exemplo, ao rotacionar no eixo Y (plano XZ), mantemos a altura ($y$) constante e recalculamos $x$ e $z$ usando essas funções. Sem o seno e o cosseno, o cubo seria apenas um conjunto estático de pontos; com eles, ganhamos a capacidade de projetar qualquer ângulo de visão mantendo a integridade geométrica do objeto.

## Implementação em Java Swing

Diferente do ambiente web, no Java Swing trabalhamos com o `Graphics2D` dentro do método `paintComponent`. Abaixo está o código completo do visualizador, que utiliza `JSlider` centralizados para permitir que o usuário explore o cubo de qualquer ângulo manualmente.

```java
package codigos.cube;

import javax.swing.*;
import java.awt.*;

public class CubeVisualizer extends JPanel {

    private static final int WIDTH = 800;
    private static final int HEIGHT = 800;
    private static final Color BACKGROUND = new Color(16, 16, 16);
    private static final Color FOREGROUND = new Color(80, 255, 80);

    // Vértices do Cubo (x, y, z)
    private static final double[][] vs = {
            { 0.25,  0.25,  0.25},
            {-0.25,  0.25,  0.25},
            {-0.25, -0.25,  0.25},
            { 0.25, -0.25,  0.25},
            { 0.25,  0.25, -0.25},
            {-0.25,  0.25, -0.25},
            {-0.25, -0.25, -0.25},
            { 0.25, -0.25, -0.25}
    };

    private static final int[][] fs = {
            {0, 1, 2, 3}, // Face frontal
            {4, 5, 6, 7}, // Face traseira
            {0, 4}, {1, 5}, {2, 6}, {3, 7} // Conexões laterais
    };

    private double dz = 1.0;
    private double manualAngleX = Math.toRadians(180);
    private double manualAngleY = Math.toRadians(180);

    public CubeVisualizer() {
        setPreferredSize(new Dimension(WIDTH, HEIGHT));
        setBackground(BACKGROUND);
    }

    // Rotação em torno do eixo Y
    private double[] rotateXZ(double[] p, double angle) {
        double x = p[0], y = p[1], z = p[2];
        double cos = Math.cos(angle), sin = Math.sin(angle);
        return new double[]{ x * cos - z * sin, y, x * sin + z * cos };
    }

    // Rotação em torno do eixo X
    private double[] rotateYZ(double[] p, double angle) {
        double x = p[0], y = p[1], z = p[2];
        double cos = Math.cos(angle), sin = Math.sin(angle);
        return new double[]{ x, y * cos - z * sin, y * sin + z * cos };
    }

    private double[] translateZ(double[] p, double dz) {
        return new double[]{p[0], p[1], p[2] + dz};
    }

    private double[] project(double[] p) {
        return new double[]{ p[0] / p[2], p[1] / p[2] };
    }

    private Point toScreen(double[] p) {
        int x = (int) ((p[0] + 1) / 2 * getWidth());
        int y = (int) ((1 - (p[1] + 1) / 2) * getHeight());
        return new Point(x, y);
    }

    @Override
    protected void paintComponent(Graphics g) {
        super.paintComponent(g);
        Graphics2D g2 = (Graphics2D) g;
        g2.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);
        g2.setColor(FOREGROUND);
        g2.setStroke(new BasicStroke(3));

        for (int[] f : fs) {
            for (int i = 0; i < f.length; i++) {
                int nextIdx = (f.length == 2 && i == 1) ? -1 : (i + 1) % f.length;
                if (nextIdx == -1) break;

                double[] p1 = vs[f[i]];
                double[] p2 = vs[f[nextIdx]];

                // Pipeline: Rotacionar (XZ -> YZ) -> Transladar -> Projetar -> Mapear para Tela
                double[] r1 = rotateYZ(rotateXZ(p1, manualAngleY), manualAngleX);
                double[] r2 = rotateYZ(rotateXZ(p2, manualAngleY), manualAngleX);

                Point s1 = toScreen(project(translateZ(r1, dz)));
                Point s2 = toScreen(project(translateZ(r2, dz)));

                g2.drawLine(s1.x, s1.y, s2.x, s2.y);
            }
        }
    }

    public void setDz(double dz) { this.dz = dz; repaint(); }
    public void setManualAngleX(double angle) { this.manualAngleX = angle; repaint(); }
    public void setManualAngleY(double angle) { this.manualAngleY = angle; repaint(); }

    public static void main(String[] args) {
        JFrame frame = new JFrame("Cubo 3D - Swing Manual Visualizer");
        CubeVisualizer visualizer = new CubeVisualizer();

        JSlider dzSlider = new JSlider(50, 300, 100);
        dzSlider.addChangeListener(e -> visualizer.setDz(dzSlider.getValue() / 100.0));

        JSlider xSlider = new JSlider(0, 360, 180);
        xSlider.addChangeListener(e -> visualizer.setManualAngleX(Math.toRadians(xSlider.getValue())));

        JSlider ySlider = new JSlider(0, 360, 180);
        ySlider.addChangeListener(e -> visualizer.setManualAngleY(Math.toRadians(ySlider.getValue())));

        JPanel controls = new JPanel(new GridLayout(1, 6));
        controls.add(new JLabel(" Distância (dz):", JLabel.RIGHT)); controls.add(dzSlider);
        controls.add(new JLabel(" Rotação X:", JLabel.RIGHT)); controls.add(xSlider);
        controls.add(new JLabel(" Rotação Y:", JLabel.RIGHT)); controls.add(ySlider);

        frame.setLayout(new BorderLayout());
        frame.add(visualizer, BorderLayout.CENTER);
        frame.add(controls, BorderLayout.SOUTH);
        frame.pack();
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.setLocationRelativeTo(null);
        frame.setVisible(true);
    }
}
```

## Dessecando o Mecanismo Manual

### Rotação Composta Estática
Nesta versão, eliminamos o fator tempo (`currentAngle`). O estado do cubo é determinado exclusivamente pelos valores dos sliders. Ao aplicar a rotação no eixo Y seguida pela rotação no eixo X, permitimos uma navegação intuitiva: o slider Y gira o cubo como um globo terrestre, e o slider X inclina a visualização.

### O Pipeline de Resposta Imediata
Como não há um timer redesenhando a tela constantemente, utilizamos o método `repaint()` dentro dos setters (`setDz`, `setManualAngleX`, `setManualAngleY`). Isso garante que a interface seja atualizada apenas quando houver interação do usuário, economizando recursos da CPU.

{: .prompt-info }
> A inicialização dos sliders em 180 graus permite que o usuário tenha "espaço" para girar o objeto em ambas as direções a partir de uma visão frontal equilibrada.

## Aplicações Práticas

Este modelo é ideal para:
- **Ferramentas de Configuração:** Onde o usuário precisa visualizar um produto de diferentes ângulos.
- **Visualização de Modelos Matemáticos:** Para entender a geometria de sólidos platônicos ou superfícies 3D.
- **Prototipagem de UI:** Para testar elementos de interface que possuam profundidade ou paralaxe.

O Java Swing, mesmo sem animação constante, demonstra uma fluidez excelente para manipulação manual de gráficos vetoriais, provando ser uma ferramenta robusta para ferramentas internas e visualizações técnicas rápidas.

## Referência

Codigo de referencia em javascript

[Tsoding formula](https://github.com/tsoding/formula)