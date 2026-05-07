# Visualizador OBJ/MTL Interativo - Documentação Técnica

## 1. Visão Geral do Projeto

O **Visualizador OBJ/MTL Interativo** é uma aplicação web desenvolvida em **HTML5, CSS3 e JavaScript vanilla** que permite carregar, visualizar e manipular modelos 3D no formato **OBJ (Wavefront Object)**, com suporte básico a materiais **MTL (Material Template Library)**.

A aplicação renderiza a malha 3D em um **Canvas 2D**, aplicando transformações geométricas por meio de matrizes 4x4 e projetando os vértices em 2D por meio de projeção isométrica ou perspectiva simplificada.

### Objetivos principais

- Carregar modelos 3D em formato `.obj`.
- Carregar materiais em formato `.mtl`.
- Aplicar cores de materiais usando principalmente a propriedade `Kd` do MTL.
- Renderizar modelos em três modos: **wireframe**, **sólido** e **wireframe + sólido**.
- Permitir rotação, escala e translação do modelo em tempo real.
- Exibir estatísticas da malha: vértices, arestas, faces e característica de Euler.
- Demonstrar conceitos fundamentais de computação gráfica sem bibliotecas externas.

---

## 2. Arquitetura do Sistema

```text
┌─────────────────────────────────────────────────────────────┐
│                    VISUALIZADOR OBJ/MTL                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────────────────────┐       ┌──────────────────────────┐  │
│  │ Interface Web       │       │ Renderização em Canvas 2D│  │
│  │                    │       │                          │  │
│  │ • Upload OBJ       │──────▶│ • Wireframe              │  │
│  │ • Upload MTL       │       │ • Sólido                 │  │
│  │ • Botões de modo   │       │ • Wireframe + sólido     │  │
│  │ • HUD              │       │ • Seleção de triângulo   │  │
│  │ • Estatísticas     │       │                          │  │
│  └────────────────────┘       └─────────────▲────────────┘  │
│                                             │               │
│                                             │               │
│  ┌──────────────────────────────────────────┴────────────┐  │
│  │              Motor 3D em JavaScript                    │  │
│  │                                                       │  │
│  │  • Parser OBJ                                        │  │
│  │  • Parser MTL                                        │  │
│  │  • Vetores 3D                                        │  │
│  │  • Matrizes 4x4                                      │  │
│  │  • Transformações geométricas                        │  │
│  │  • Projeção isométrica/perspectiva                   │  │
│  │  • Backface culling                                  │  │
│  │  • Ordenação por profundidade                        │  │
│  └──────────────────────────────────────────▲────────────┘  │
│                                             │               │
│  ┌──────────────────────────────────────────┴────────────┐  │
│  │                  Estruturas de Dados                   │  │
│  │                                                       │  │
│  │  • positions[]: vértices 3D                           │  │
│  │  • normals[]: normais do OBJ                          │  │
│  │  • uvs[]: coordenadas UV lidas, mas não renderizadas  │  │
│  │  • faces[]: faces originais do OBJ                    │  │
│  │  • triangles[]: faces trianguladas para renderização  │  │
│  │  • materials: mapa de materiais MTL                   │  │
│  │  • modelMatrix: matriz de transformação do modelo     │  │
│  │  • stats: V, E, F e Euler                             │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Componentes principais

1. **Interface de usuário**
   - Painel lateral com upload de arquivos, botões, estatísticas e instruções.
   - Área principal com o Canvas 2D.
   - HUD com projeção, modo de renderização e ferramenta ativa.

2. **Motor 3D**
   - Parser para arquivos OBJ.
   - Parser para arquivos MTL.
   - Implementação manual de vetores e matrizes.
   - Transformações de rotação, escala e translação.
   - Projeção 3D para 2D.

3. **Sistema de renderização**
   - Renderização em Canvas 2D.
   - Modos wireframe, sólido e híbrido.
   - Backface culling baseado na normal da face transformada.
   - Ordenação por profundidade usando algoritmo do pintor.
   - Iluminação simples do tipo Lambert.

4. **Validação topológica**
   - Contagem de vértices, arestas e faces.
   - Cálculo da característica de Euler `V - E + F`.
   - Indicação visual quando o resultado é igual a 2.

---

## 3. Decisões Técnicas

### 3.1 JavaScript vanilla + Canvas 2D

A aplicação foi implementada sem bibliotecas externas, usando apenas recursos nativos do navegador.

#### Justificativas

- **Portabilidade**: o projeto funciona como um único arquivo HTML.
- **Sem build system**: não depende de Node.js, bundlers ou instalação de pacotes.
- **Foco educacional**: deixa explícitos os cálculos de vetores, matrizes, projeção e renderização.
- **Controle total**: todos os passos do pipeline gráfico são implementados manualmente.

#### Trade-offs

- O Canvas 2D é menos eficiente que WebGL para modelos grandes.
- Não há aceleração gráfica 3D nativa.
- Recursos como texturas, sombras avançadas e câmera completa precisam ser implementados manualmente.

---

### 3.2 Matrizes 4x4

As transformações geométricas são representadas por matrizes 4x4. Isso permite compor rotação, escala e translação usando multiplicação de matrizes.

Exemplo de matriz de translação:

```javascript
[1, 0, 0, x,
 0, 1, 0, y,
 0, 0, 1, z,
 0, 0, 0, 1]
```

A aplicação mantém uma matriz principal chamada `modelMatrix`, que acumula as transformações aplicadas ao modelo.

---

### 3.3 Projeções disponíveis

A aplicação possui dois modos de projeção.

#### Projeção isométrica

- Usa uma matriz base com rotações nos eixos X e Y.
- Não aplica efeito de profundidade por distância.
- É adequada para visualizar a estrutura do modelo de forma mais técnica.

#### Projeção perspectiva simplificada

- Usa uma distância fixa de câmera (`cameraZ`).
- Aplica um fator de escala baseado na profundidade do ponto.
- Não possui controle de FOV pela interface.
- Utiliza clamp de profundidade para evitar valores muito pequenos durante a projeção.

---

### 3.4 Parsing de OBJ e MTL

#### Parser OBJ

O parser OBJ suporta:

- Vértices: `v x y z`.
- Normais: `vn x y z`.
- Coordenadas UV: `vt u v`.
- Faces nos formatos:
  - `f v1 v2 v3`
  - `f v1//vn1 v2//vn2 v3//vn3`
  - `f v1/vt1/vn1 v2/vt2/vn2 v3/vt3/vn3`
  - `f v1/vt1 v2/vt2 v3/vt3`
- Índices negativos, conforme a especificação OBJ.
- Referências a materiais por `usemtl`.
- Referências a bibliotecas de material por `mtllib`.

Observação: o nome indicado em `mtllib` é registrado e exibido no log, mas o arquivo MTL precisa ser carregado manualmente pelo usuário no campo correspondente.

#### Parser MTL

O parser MTL suporta:

- `newmtl`
- `Kd`
- `Ka`
- `Ks`
- `Ns`

Na renderização atual, a cor base do material vem principalmente de `Kd`. As propriedades `Ka`, `Ks` e `Ns` são lidas e armazenadas, mas não são usadas em um modelo de iluminação especular completo.

---

## 4. Bibliotecas e Componentes

| Componente | Tecnologia | Justificativa |
|---|---|---|
| Estrutura | HTML5 | Define a interface e os controles. |
| Estilo | CSS3 | Cria o layout responsivo, painel lateral e HUD. |
| Lógica | JavaScript ES6+ | Implementa parser, motor 3D e interação. |
| Renderização | Canvas 2D API | Desenha triângulos e linhas no navegador. |
| Matemática | Implementação manual | Vetores, matrizes, normais e projeção. |

### Dependências externas

O projeto não usa dependências externas:

- Sem Three.js.
- Sem Babylon.js.
- Sem React, Vue ou Angular.
- Sem bibliotecas de matemática.
- Sem processo de build.

---

## 5. Funcionamento Interno

### 5.1 Fluxo de carregamento

1. O usuário seleciona um arquivo `.obj`.
2. Opcionalmente, seleciona um arquivo `.mtl`.
3. O MTL é lido primeiro, se existir.
4. O OBJ é parseado, associando faces aos materiais ativos.
5. As faces com mais de três vértices são trianguladas por fan triangulation.
6. O modelo é centralizado no centroide.
7. O modelo é normalizado para caber melhor na tela.
8. As estatísticas topológicas são calculadas.
9. O loop de renderização passa a desenhar o modelo no Canvas.

---

### 5.2 Triangulação

O Canvas desenha polígonos, mas a aplicação converte faces de 4 ou mais vértices em triângulos para simplificar o pipeline de renderização.

A técnica usada é **fan triangulation**:

```text
Face original:  v0, v1, v2, v3, v4
Triângulos:     (v0, v1, v2), (v0, v2, v3), (v0, v3, v4)
```

Essa abordagem é simples e funciona bem para faces convexas. Em faces côncavas, pode gerar resultados visuais incorretos.

---

### 5.3 Centralização e normalização

Após o carregamento, o modelo é centralizado no próprio centroide. Em seguida, é normalizado pelo maior raio encontrado entre os vértices.

Isso evita que modelos muito grandes ou deslocados apareçam fora da tela logo após o carregamento.

---

### 5.4 Normais

Se o arquivo OBJ fornecer normais (`vn`), a aplicação calcula uma média simples das normais dos vértices da face triangular.

Se o OBJ não fornecer normais, a normal da face é calculada por produto vetorial:

```javascript
normal = cross(p1 - p0, p2 - p0)
```

Essa normal é usada para backface culling e iluminação simples.

---

### 5.5 Backface culling

A aplicação calcula a normal da face no espaço transformado e verifica o sinal da componente `z`.

Faces com normal voltada para trás são ignoradas na renderização. Isso reduz a quantidade de triângulos desenhados e melhora a legibilidade visual do modelo.

---

### 5.6 Ordenação por profundidade

Como o Canvas 2D não possui depth buffer, a aplicação usa uma estratégia simples de ordenação por profundidade.

Os triângulos são ordenados pela profundidade média dos seus vértices, e os mais distantes são renderizados primeiro.

Essa técnica é conhecida como **algoritmo do pintor**. Ela funciona bem para muitos modelos simples, mas pode falhar em geometrias complexas com interseções ou sobreposições ambíguas.

---

### 5.7 Iluminação

A iluminação é simplificada e baseada em Lambert:

```javascript
lambert = max(0.12, dot(normal, lightDir))
```

O valor calculado escurece ou clareia a cor `Kd` do material. A aplicação não implementa sombras, reflexos, brilho especular real ou múltiplas luzes.

---

## 6. Interface e Controles

### Upload de arquivos

- **Arquivo .OBJ**: carrega a geometria do modelo.
- **Arquivo .MTL / objetos.mtl**: carrega os materiais usados pelo modelo.

### Botões de renderização

| Botão | Função |
|---|---|
| `W Wireframe` | Ativa renderização apenas com arestas. |
| `S Sólido` | Ativa renderização preenchida. |
| `Wire + Sólido` | Combina preenchimento e arestas. |
| `P Projeção` | Alterna entre projeção isométrica e perspectiva. |

### Teclado e mouse

| Controle | Função |
|---|---|
| `W` | Ativa modo wireframe. |
| `B` | Ativa modo wireframe + sólido. |
| `P` | Alterna entre projeção isométrica e perspectiva. |
| `R` | Alterna a ferramenta de rotação. |
| `R` ativo + `X`, `Y` ou `Z` | Rotaciona no eixo correspondente. |
| `Shift` + eixo | Inverte o sentido da rotação. |
| `S` | Alterna a ferramenta de escala. |
| `S` ativo + setas | Aumenta ou reduz a escala uniformemente. |
| `T` | Alterna a ferramenta de translação. |
| `T` ativo + setas | Move o modelo no plano da tela. |
| `Esc` | Reseta as transformações e desativa a ferramenta atual. |
| Arrastar com mouse | Rotaciona o modelo continuamente. |
| Scroll do mouse | Aproxima ou afasta o modelo por escala. |
| Clique no modelo | Seleciona e destaca um triângulo visível. |

> Observação: no código atual, a tecla `S` é usada principalmente para a ferramenta de escala. Para ativar diretamente o modo sólido, o caminho mais claro é usar o botão **S Sólido** da interface.

---

## 7. Estatísticas e Fórmula de Euler

A aplicação calcula e exibe:

- **V**: quantidade de vértices do OBJ.
- **E**: quantidade de arestas únicas derivadas das faces originais.
- **F**: quantidade de faces originais do OBJ.
- **Euler**: resultado de `V - E + F`.

Para poliedros fechados, simples e com topologia equivalente a uma esfera, espera-se:

```text
V - E + F = 2
```

Quando o resultado é igual a 2, a interface exibe uma mensagem de sucesso. Caso contrário, exibe um aviso, pois modelos abertos, com buracos, múltiplos componentes ou topologia diferente podem produzir outro valor.

### Implementação conceitual

```javascript
const edgeSet = new Set();

for (const face of obj.faces) {
  for (let i = 0; i < face.verts.length; i++) {
    const a = face.verts[i].v;
    const b = face.verts[(i + 1) % face.verts.length].v;
    const lo = Math.min(a, b);
    const hi = Math.max(a, b);
    edgeSet.add(`${lo}-${hi}`);
  }
}

const euler = vertices.length - edgeSet.size + faces.length;
```

---

## 8. Modelo Inicial de Demonstração

Quando a página é aberta, a aplicação carrega automaticamente um **icosaedro de demonstração** embutido no próprio JavaScript.

Esse modelo inicial permite testar imediatamente:

- Renderização sólida.
- Projeção isométrica.
- Materiais básicos.
- Rotação por mouse.
- Cálculo de Euler.

O material de demonstração usa um MTL simples chamado `bronze`, também embutido no código.

---

## 9. Otimizações Implementadas

1. **Backface culling**: evita desenhar faces voltadas para trás.
2. **Ordenação por profundidade**: renderiza faces mais distantes antes das mais próximas.
3. **Triangulação prévia das faces**: simplifica o processo de desenho.
4. **Centralização e normalização do modelo**: facilita a visualização inicial.
5. **Uso de `requestAnimationFrame`**: sincroniza o loop de renderização com o navegador.
6. **Clamp na projeção perspectiva**: evita divisões por profundidades muito pequenas.

---

## 10. Limitações Conhecidas

1. **Renderização apenas com Canvas 2D**: não usa WebGL nem depth buffer real.
2. **Sem texturas**: coordenadas UV são lidas, mas não são aplicadas na renderização.
3. **Sem carregamento automático do MTL referenciado**: o usuário precisa selecionar o arquivo MTL manualmente.
4. **Iluminação simplificada**: usa apenas sombreamento difuso básico.
5. **Sem sombras dinâmicas**.
6. **Sem brilho especular real**, mesmo que `Ks` e `Ns` sejam lidos.
7. **Sem exportação**: não salva o modelo transformado.
8. **Algoritmo do pintor pode falhar** em geometrias complexas ou auto-intersectantes.
9. **Triangulação fan pode falhar visualmente** em faces côncavas.
10. **Performance limitada em modelos grandes**, por depender de Canvas 2D e JavaScript puro.

---

## 11. Conclusão

O **Visualizador OBJ/MTL Interativo** demonstra como construir um pipeline 3D básico usando apenas recursos nativos do navegador.

Mesmo sem WebGL ou bibliotecas externas, o projeto cobre etapas importantes da computação gráfica:

- Leitura de arquivos OBJ e MTL.
- Organização de vértices, faces, normais e materiais.
- Transformações geométricas com matrizes 4x4.
- Projeção de pontos 3D para 2D.
- Renderização em Canvas 2D.
- Backface culling.
- Ordenação por profundidade.
- Validação topológica por característica de Euler.
