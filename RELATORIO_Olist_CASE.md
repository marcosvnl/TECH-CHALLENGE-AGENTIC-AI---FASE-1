# Relatório Executivo — Olist Intelligent Marketplace
### Tech Challenge — Fase 1 | Fundamentos de IA e Agentic AI
**v2 — Diagnóstico de Negócio + Priorização para IA Agentic** (mapa de agentes, arquitetura conceitual e prompts serão adicionados na v3)

---

## 1. Sumário Executivo

A Olist processou aproximadamente 100 mil pedidos entre 2016 e 2018, conectando pequenos e médios lojistas aos maiores marketplaces do Brasil. A análise integrada das oito bases públicas do dataset (pedidos, itens, pagamentos, avaliações, produtos, vendedores, clientes e geolocalização) revela que os problemas mais graves da operação não estão na aquisição de clientes — a plataforma converte bem — e sim na experiência pós-compra e na retenção. A nota média da plataforma é 4,09, mas a taxa de recompra é de apenas 3%, e um conjunto de causas estruturais e operacionais recorrentes explica a maior parte das avaliações negativas.

Esta análise identificou e quantificou dez frentes de problema, cobrindo cerca de 86% de todas as avaliações nota 1-2 registradas na base. Cada uma dessas frentes está ancorada em evidência de dado (não em suposição). A partir dessas dez frentes, cinco foram priorizadas para atuação com Agentes de IA — selecionadas pela combinação de impacto demonstrável, evidência quantitativa robusta e potencial concreto de automação inteligente. Este documento é a base sobre a qual, na próxima etapa, serão desenhados os agentes de IA responsáveis por endereçá-las.

---

## 2. Contexto do Negócio

A Olist opera como uma camada de intermediação entre pequenos e médios lojistas (o "empresário" que se cadastra na plataforma) e os grandes marketplaces do país. O lojista paga uma assinatura para acessar a reputação e a infraestrutura de exposição da Olist, mais uma comissão sobre cada venda. A Olist é a marca que o consumidor final enxerga; a logística e o cumprimento do pedido, porém, são responsabilidade do lojista.

Essa arquitetura cria uma tensão estrutural: a reputação da Olist é um agregado do desempenho de milhares de vendedores independentes, sobre os quais ela tem controle indireto. Um vendedor que vende sem estoque real, demora para despachar ou erra a quantidade de um pedido não prejudica apenas a si mesmo — prejudica a marca Olist como um todo, e é isso que a análise a seguir evidencia com dados.

---

## 3. Sobre a Base de Dados e Metodologia

A análise integrou as tabelas `orders`, `order_items`, `order_payments`, `order_reviews`, `products`, `sellers`, `customers` e `geolocation`, relacionadas por `order_id`, `customer_id`/`customer_unique_id`, `product_id`, `seller_id` e `zip_code_prefix`. A base cobre pedidos realizados entre 2016 e 2018, com foco nos ~97 mil pedidos com status `delivered`.

A abordagem seguida foi: (1) formular hipóteses de negócio testáveis para cada desafio citado no briefing (experiência do cliente, logística, retenção, reviews, eficiência operacional); (2) testar cada hipótese diretamente na base, comparando grupos e isolando variáveis de confusão sempre que possível (por exemplo, separar o efeito do atraso do efeito da categoria do produto); (3) quantificar magnitude e abrangência de cada achado antes de reportá-lo. Duas hipóteses plausíveis foram testadas e descartadas por falta de sustentação nos dados: tempo de aprovação de pagamento não se traduz em mais atraso final, e a quantidade de fotos/descrição do produto não afeta a nota de forma geral (só em categorias sensoriais/têxteis específicas).

Como validação adicional de rigor, foi feita uma segunda passada isolando estatisticamente todas as causas já identificadas para garantir que nenhum problema relevante estivesse sendo mascarado por outro. O resultado: 86% das avaliações negativas são explicadas pelas dez frentes descritas abaixo; o restante é cauda longa de reclamações dispersas, sem concentração que justifique uma frente própria — com uma exceção notável, tratada na seção 4.6.

---

## 4. Diagnóstico Operacional

### 4.1 Atraso de entrega — o maior driver isolado de insatisfação

O atraso é o fator com maior impacto isolado na nota do cliente. Pedidos atrasados recebem nota média 2,27, contra 4,29 dos pedidos no prazo — uma queda de quase 2 pontos inteiros numa escala de 1 a 5. Entre os pedidos atrasados, entre 53,7% e 62,4% (a depender do corte de nota) recebem nota 1. A taxa geral de atraso varia entre 6,8% e 8,1%, dependendo da definição usada (comparação direta com a data estimada vs. tolerância adicional).

Dois achados qualificam esse problema além do óbvio: primeiro, o prazo estimado é excessivamente conservador — a Olist entrega, em média, 11 a 12 dias antes do prazo prometido, e 85% dos pedidos chegam com 5 ou mais dias de antecedência. Isso protege a nota, mas esconde o desempenho logístico real e reduz a competitividade do prazo exibido no checkout. Segundo, a variabilidade é tão relevante quanto a média: o desvio padrão do tempo de entrega é de 9,6 dias, indicando uma operação inconsistente, não apenas lenta.

Houve ainda um evento isolado relevante: na janela de fevereiro–março de 2018, a taxa de atraso saltou para 26,3%, superando o pico da própria Black Friday (12,4%, novembro de 2017). Nesse episódio, o tempo de despacho dos vendedores permaneceu estável — a causa foi inteiramente da malha de transporte terceirizada, não da operação da Olist ou dos lojistas.

### 4.2 Desempenho de vendedores — a alavanca mais controlável

Decompondo o tempo total de entrega (cerca de 12,5 dias em média): aprovação de pagamento (10,3h), despacho pelo vendedor (2,8 dias) e transporte (9,3 dias, majoritariamente fora do controle direto da Olist). Apesar de ser a menor fatia, o despacho é a etapa mais acionável pela empresa. A correlação entre tempo médio de despacho e nota do vendedor é de -0,45 — forte.

O achado mais contundente da análise: em apenas 4,5% dos pedidos o vendedor despacha depois do prazo interno combinado (`shipping_limit_date`) — mas quando isso acontece, a nota cai de 4,19 para 3,45 e a taxa de atraso final na ponta do cliente salta de 5,7% para 30,1%, um efeito de cinco vezes. É baixo volume, mas altíssima causalidade, e inteiramente sob controle de um SLA — a alavanca de maior retorno por esforço identificada neste diagnóstico.

Some-se a isso um problema de concentração de risco: os 10% maiores vendedores respondem por cerca de 68% do GMV da plataforma. Em pelo menos um caso, um único vendedor concentra 73% de todo o volume de uma categoria inteira (móveis de escritório), com nota própria pior que a média da categoria — o que parece estatisticamente "um problema de categoria de produto" é, na prática, risco concentrado em um único fornecedor.

### 4.3 Ruptura de estoque — venda sem disponibilidade real

Do total de pedidos cancelados, 77,4% ocorrem **depois** da aprovação do pagamento — ou seja, o lojista vendeu um produto que não tinha em estoque. Desses, 84,5% nunca chegaram a ser despachados para a transportadora. Um grupo pequeno de nove vendedores concentra cinco ou mais cancelamentos cada, indicando que não se trata de eventos aleatórios, e sim de um problema sistêmico de controle de inventário em uma parcela específica da base de lojistas. As categorias mais afetadas são as de alta rotatividade — esportes/lazer, utilidades domésticas e informática.

### 4.4 Pagamento — atrito estrutural do boleto

73,9% dos pedidos são pagos em cartão de crédito (média de 3,5 parcelas), 19% em boleto. A diferença de velocidade de aprovação entre os dois métodos é enorme: mediana de 16 minutos no cartão contra 29 horas no boleto. Esse atraso gera um efeito em cascata mensurável: pedidos com aprovação de pagamento mais lenta (48–100h) têm taxa de atraso final de 8,2%, contra 6,4% nos de aprovação rápida — um gradiente real, embora moderado. Em 142 pedidos, o cancelamento ocorreu com o pagamento nunca chegando a ser aprovado. Parcelamentos muito longos (11 ou mais vezes) também correlacionam com nota mais baixa (3,87 vs. 4,12 à vista), um sinal secundário de atrito financeiro do comprador.

### 4.5 Catálogo e qualidade de produto

Item trocado ou entregue errado é a pior experiência isolada registrada na base: nota média 1,91, contra 4,09 geral. Descrição incorreta do produto tem impacto um pouco menor, mas ainda relevante: nota 3,09. Isolando especificamente os pedidos que chegaram **dentro do prazo** (eliminando qualquer efeito logístico), ainda assim cerca de 4,8% de todas as avaliações negativas mencionam defeito, dano ou divergência do produto recebido — comprovando que existe um problema de qualidade/fidelidade de catálogo independente da logística.

Um padrão adicional, mais sutil e não coberto por nenhuma categoria anterior: envio parcial ou quantidade divergente. São casos em que o vendedor é o correto e o produto é o correto, mas a quantidade entregue é menor que a comprada — "comprei 5, vieram 3", "faltou uma peça do kit". Esse padrão aparece em cerca de 2% das avaliações negativas e é operacionalmente distinto de item errado (é erro de separação/picking no estoque do vendedor, não de cadastro do anúncio).

### 4.6 Fragmentação de pedidos — o custo escondido de comprar mais de uma coisa

Dois fatores estruturais do pedido, independentes do atraso, reduzem fortemente a satisfação. Pedidos com múltiplos itens do mesmo vendedor têm a nota caindo de 4,21 (1 item) para 3,44 (5 itens) — mesmo com tempo de entrega e taxa de atraso estáveis; afeta 10,2% dos pedidos. Pedidos que reúnem itens de dois ou mais vendedores são ainda mais graves: a nota despenca para 2,89 (2 vendedores) e 2,35 (3 vendedores), apesar de esses pedidos chegarem, em média, **mais rápido** que os de vendedor único (efeito estatisticamente muito robusto). Comentários desses pedidos citam recebimento parcial ou incompleto em 32,5% dos casos, contra 3,2% em pedidos de vendedor único — dez vezes mais frequente. O problema aqui não é velocidade, é comunicação e rastreamento de remessas parciais.

### 4.7 Frete — punição estrutural de quem compra mais

Este é um achado de política de precificação, não de percepção do cliente. O frete é cobrado por item, não por pedido: ele representa, em média, 20,6% do valor total em pedidos de 1 item, subindo para 26,6% em pedidos de 5 itens. O efeito na satisfação só aparece quando o frete alto se combina com múltiplos itens — nesse caso a nota cai para 3,57. Quando o frete é proporcionalmente alto mas o pedido é de 1 item só (comum em produtos baratos), a nota não se move (4,16, igual à média geral) — o cliente aceita, porque sabia o valor antes de comprar. O problema não é "frete caro" isoladamente: é frete pago em duplicidade dentro do mesmo pedido. Um caso real documentado na base ilustra isso com clareza: pedido de 21 itens de um mesmo vendedor, produto total de R$ 31,80 e frete total de R$ 164,37 (84% do valor pago) — o cliente reclamou textualmente de "frete abusivo" na avaliação.

### 4.8 Geografia — a malha logística não é uniforme

São Paulo concentra cerca de 60% dos vendedores da plataforma e 64% da receita. Isso cria dependência estrutural de uma única origem geográfica. O impacto na entrega varia fortemente por região de destino: o Nordeste tem 12,7% de atraso e nota média 3,97, contra 6,1% e 4,18 no Sudeste. O Norte tem o maior tempo de trânsito absoluto — 22 dias partindo de SP, quase três vezes o tempo SP→SP (7,5 dias). Um achado mais fino: o Rio de Janeiro tem penalidade própria mesmo controlando pela distância — na mesma faixa de 300 a 700 km, pedidos para o RJ atrasam 12,7% contra 4,8% de outros estados equidistantes, sugerindo um problema específico de malha urbana/portuária, e não de distância geográfica pura. De forma mais simples e imediatamente acionável: pedidos interestaduais (vendedor e cliente em estados diferentes) já têm o risco de atraso quase dobrado (9,0% vs. 6,0%) — uma variável conhecida no momento da compra, antes mesmo de qualquer roteamento específico.

### 4.9 Retenção e recompra — um problema de negócio à parte

Apenas 3% dos clientes únicos realizam mais de uma compra em dois anos de operação; 97% são "clientes de viagem única". O achado estatístico mais importante desta seção é que satisfação e recompra estão praticamente desacopladas: a correlação entre nota da primeira compra e probabilidade de recompra é de apenas 0,006 — estatisticamente nula. Isso significa que melhorar a experiência de entrega (seções 4.1 a 4.8) não vai, sozinho, resolver o problema de retenção. São dois problemas de negócio distintos, que exigem soluções diferentes: um cuida da experiência pós-venda, outro precisa de reativação proativa. A recompra correlaciona mais com o tipo de categoria (produtos de reposição/consumo chegam a 18,1% de recompra, muito acima da média) e com uma janela de tempo específica — a mediana entre a primeira e a segunda compra é de 28 dias; passada essa janela, a probabilidade de reativação cai.

### 4.10 Comunicação e reputação — os limites do que os dados conseguem mostrar

Apenas 41,3% das avaliações incluem comentário em texto, o que já limita qualquer leitura de causa-raiz em escala. Mais relevante ainda: 8,1% de todas as notas baixas não têm nenhum texto explicativo — são casos de insatisfação "silenciosa", sem nenhuma pista textual da causa. É uma limitação estrutural a considerar no desenho de qualquer solução baseada em análise de linguagem: ela vai sistematicamente deixar de capturar cerca de 1 em cada 12 clientes insatisfeitos.

Um padrão adicional, e potencialmente sensível, foi encontrado nos textos: 1.718 pedidos aparecem como "entregues" no sistema, mas o cliente afirma explicitamente "não recebi o produto" (nota média 1,45). Em parte relevante desses casos, a reclamação foi escrita depois da data de entrega registrada — o que sugere contestação direta do registro de entrega, e não apenas impaciência. Isso pode indicar falha na prova de entrega da transportadora (assinatura/foto), um problema de integridade de dado distinto do atraso simples.

---

## 5. Os Cinco Problemas Prioritários para Atuação com IA

A partir dos dez achados do diagnóstico operacional e considerando os desafios centrais que a Olist enfrenta, que são: experiência do cliente, logística, retenção, reviews, eficiência operacional, automação, decisão baseada em dados e escalabilidade. Portanto, foram selecionados cinco problemas que combinam três critérios de priorização: (1) impacto demonstrável no negócio, (2) evidência quantitativa robusta nos dados e (3) potencial concreto de atuação com Agentes de IA. Estes são os problemas sobre os quais a arquitetura de agentes de IA irá apoiar a Olist a enfrentar seus desafios de negócio e melhorar os indicadores de negócio e saúde operacional.

Os cinco problemas não são independentes — eles formam uma cadeia de causa → impacto → ação que segue uma progressão definida: **excelência na operação → melhora na experiência dos clientes → retenção e fidelização**.

1. **Desempenho dos vendedores** — a causa operacional mais controlável pela Olist;
2. **Atraso de entrega** — consequência direta de falhas no despacho e na malha logística, diretamente percebido pelo cliente;
3. **Catálogo e qualidade de produto** — problema de experiência que independe da logística, também diretamente percebido pelo cliente;
4. **Comunicação e reputação** — a capacidade de transformar o feedback do cliente em sinal operacional, inclusive identificando anomalias como "entregue, mas não recebido";
5. **Retenção e recompra** — o resultado econômico final: mesmo que a operação funcione melhor, o diagnóstico mostra que satisfação e recompra estão praticamente desacopladas (correlação 0,006), portanto a retenção exige uma estratégia própria.

Cada problema priorizado pode resultar em um agente ou capacidade agentic claramente justificável por dados — essa é a ponte entre o diagnóstico quantitativo e a arquitetura de IA.

### 5.1 Desempenho dos vendedores

É uma das maiores oportunidades de melhoria porque está mais diretamente sob controle da Olist do que o desempenho da transportadora, e é a causa-raiz operacional que alimenta os problemas seguintes na cadeia. Os dados confirmam: quando o vendedor atrasa o despacho (4,5% dos pedidos), a taxa de atraso na ponta do cliente salta de 5,7% para 30,1%, um efeito de 5x. Soma-se a concentração de risco: 10% dos vendedores respondem por 68% do GMV. Agentes de IA podem atuar aqui como monitores de SLA em tempo real, com alertas precoces e scoring de risco que identifiquem deterioração antes que o cliente seja impactado.

### 5.2 Atraso de entrega

É o maior indicador isolado de insatisfação do cliente na plataforma e afeta diretamente a experiência pós-compra. Quando há atraso, a nota média cai de 4,29 para 2,27, e a taxa geral oscila entre 6,8% e 8,1% da base com variabilidade de 9,6 dias de desvio padrão. Agentes preditivos podem atuar na antecipação de risco de atraso no momento do despacho, em alertas proativos ao consumidor e na reclassificação dinâmica de promessas de prazo com base em padrões históricos de rota e vendedor.

### 5.3 Catálogo e qualidade de produto

Ataca problemas graves de experiência que independem da logística e podem ser tratados com IA de forma direta. Item errado é a pior experiência isolada da base (nota 1,91), e mesmo em pedidos entregues no prazo, 4,8% das avaliações negativas citam defeito, dano ou divergência. Agentes de IA podem atuar na auditoria automatizada de catálogo, na validação cruzada entre anúncio e reclamações nos reviews, e em alertas que identifiquem anúncios com alto risco de divergência antes da venda, reduzindo o problema na origem.

### 5.4 Comunicação e reputação

É particularmente relevante para o escopo de IA Generativa e Agentic AI, pois transforma reviews e reclamações em inteligência operacional acionável. Hoje, 41,3% das avaliações incluem texto, 8,1% das notas baixas são "silenciosas", e 1.718 pedidos registrados como entregues são contestados pelo cliente. Agentes de IA Generativa podem realizar classificação automática de causa-raiz a partir do texto livre, detectar padrões emergentes de reclamação e identificar contestações de entrega que exijam investigação, transformando dados não-estruturados em decisões operacionais em escala.

### 5.5 Retenção e recompra

É provavelmente o maior problema econômico de longo prazo da Olist, e o resultado final da cadeia. Com apenas 3% de taxa de recompra em dois anos, a plataforma demonstra capacidade de conversão mas quase nenhuma de retenção. Satisfação e recompra são estatisticamente desacopladas (correlação de 0,006), ou seja, resolver os problemas de experiência é necessário mas não suficiente. A recompra responde a gatilhos específicos: categorias de reposição (até 18,1% de retorno) e janela de 28 dias como mediana entre compras. Agentes de IA podem atuar na segmentação de clientes por propensão de retorno, no disparo de comunicações no momento certo e na recomendação contextualizada de produtos complementares em escala.

### E os outros itens do diagnóstico ?

Os demais problemas identificados no diagnóstico (ruptura de estoque, atrito de boleto, fragmentação de pedidos, política de frete e desigualdade geográfica) não são descartados. São tratados como **prioridade secundária ou subproblemas** dos cinco acima: ruptura de estoque é um caso específico de desempenho do vendedor (5.1); atrito do boleto e fragmentação de pedidos são fatores agravantes do atraso e da experiência (5.2/5.3); política de frete é endereçável por regra de negócio; e desigualdade geográfica é uma restrição de infraestrutura que condiciona a gravidade do atraso regional. Todos podem ser incorporados como sub-objetivos dos agentes principais à medida que a solução evolui.

---

## 6. Limitações da Análise

A base cobre o período de 2016 a 2018 e pode não refletir integralmente a operação atual da Olist. A leitura de causa-raiz via texto de review tem cobertura parcial (41,3% das avaliações têm comentário), e 8,1% das notas baixas não têm nenhuma explicação textual disponível. Correlações identificadas (por exemplo, parcelamento longo e nota mais baixa) indicam associação, não necessariamente causalidade direta, e foram reportadas como sinais moderados, não como conclusões definitivas.

---

## 7. Próximos Passos (v3 deste documento)

### 7.1 Os cinco problemas a endereçar com IA Agentic

A partir do diagnóstico operacional (seção 4) e da matriz de priorização (seção 5), foram identificados **cinco problemas que serão endereçados com IA Agentic**. Eles formam uma cadeia de causa → impacto → ação orientada a três resultados de negócio: excelência operacional, melhora na experiência do cliente e retenção/fidelização.

| # | Problema | Evidência-chave | Resultado esperado com IA |
|---|----------|-----------------|---------------------------|
| 1 | **Desempenho dos vendedores** | 4,5% dos pedidos com despacho fora do SLA → taxa de atraso salta de 5,7% para 30,1% (efeito de 5×) | Monitor de SLA em tempo real + scoring de risco de vendedor |
| 2 | **Atraso de entrega** | Nota cai de 4,29 → 2,27 em pedidos atrasados; variabilidade de 9,6 dias de desvio padrão | Predição de risco de atraso + alertas proativos ao consumidor |
| 3 | **Catálogo e qualidade de produto** | Item errado: nota 1,91 (pior experiência da base); 4,8% das avaliações negativas citam divergência mesmo em pedidos no prazo | Auditoria automatizada de catálogo + alertas de anúncio de alto risco |
| 4 | **Comunicação e reputação** | 1.718 pedidos "entregues" contestados pelo cliente (nota 1,45); 8,1% das notas baixas sem texto explicativo | Classificação de causa-raiz em reviews + detecção de contestações de entrega |
| 5 | **Retenção e recompra** | 97% dos clientes compram apenas uma vez; correlação satisfação × recompra = 0,006 (nula) | Segmentação de propensão de retorno + comunicação no gatilho correto (janela de 28 dias) |

Estes cinco problemas são o escopo direto da v3 deste documento, onde cada um será traduzido em um agente de IA com objetivo, usuários envolvidos, entradas/saídas e prompts de referência.

---

### 7.2 Evidências analíticas dos cinco problemas

> **Orientação para o grupo de estudo:** Esta seção serve como guia editorial para enriquecer o **Capítulo 5** com gráficos analíticos que representem visualmente cada um dos cinco problemas priorizados. Os blocos a seguir indicam quais dados devem ser plotados, qual comparação deve ser destacada e qual a fonte de dado correspondente no dataset Olist. Os gráficos definitivos devem ser produzidos pelo grupo e inseridos nas subseções 5.1 a 5.5, substituindo ou complementando o texto descritivo existente.

---

#### Referência para gráfico — Problema 1: Desempenho dos vendedores

```
Nota média do pedido por cumprimento do SLA de despacho do vendedor
────────────────────────────────────────────────────────────────────
Despacho no prazo (95,5% dos pedidos)   ██████████████████  4,19
Despacho fora do prazo (4,5%)           ████████████        3,45
────────────────────────────────────────────────────────────────────

Taxa de atraso na entrega ao cliente por cumprimento do SLA
────────────────────────────────────────────────────────────────────
Despacho no prazo    ████  5,7%
Despacho fora prazo  █████████████████████████████████  30,1%
                     ← efeito de 5× →
────────────────────────────────────────────────────────────────────
Fonte: cruzamento orders × order_items × shipping_limit_date
```

**Leitura:** Um grupo pequeno (4,5% dos pedidos) com despacho fora do SLA do vendedor multiplica por cinco a taxa de atraso percebida pelo cliente. É a alavanca de maior retorno por esforço do diagnóstico — baixo volume, altíssima causalidade.

---

#### Referência para gráfico — Problema 2: Atraso de entrega

```
Nota média por status de entrega
────────────────────────────────────────────────────────────────────
No prazo        ████████████████████████████████████████  4,29
Atrasado        ███████████                               2,27
────────────────────────────────────────────────────────────────────

Distribuição de notas 1–2 em pedidos atrasados
────────────────────────────────────────────────────────────────────
Nota 1 ou 2     ██████████████████████████████  53,7% – 62,4%
Nota 3–5        ████████████████████            37,6% – 46,3%
────────────────────────────────────────────────────────────────────

Taxa de atraso por período (pico identificado)
────────────────────────────────────────────────────────────────────
Operação regular     ██  6,8% – 8,1%
Black Friday 2017    ███████  12,4%
Fev–Mar 2018         █████████████████  26,3%  ← causa: transportadora
────────────────────────────────────────────────────────────────────
Fonte: orders (order_delivered_customer_date vs. order_estimated_delivery_date)
```

**Leitura:** A queda de quase 2 pontos na nota média (de 4,29 para 2,27) é o maior efeito isolado registrado na base. O pico de fevereiro–março de 2018 (26,3%) foi causado inteiramente pela malha de transporte — não pelo vendedor —, o que reforça a necessidade de predição e comunicação proativa.

---

#### Referência para gráfico — Problema 3: Catálogo e qualidade de produto

```
Nota média por tipo de problema relatado no review
────────────────────────────────────────────────────────────────────
Item errado / trocado       ████████  1,91   ← pior experiência
Descrição incorreta         ████████████████ 3,09
Defeito / dano / divergência ███████████████ ~2,8 (entregues no prazo)
Média geral da plataforma   ████████████████████  4,09
────────────────────────────────────────────────────────────────────

% de avaliações negativas com problema de catálogo (pedidos no prazo)
────────────────────────────────────────────────────────────────────
Defeito, dano ou divergência   4,8%  ████
Envio parcial / qtd errada     2,0%  ██
────────────────────────────────────────────────────────────────────
Fonte: order_reviews (filtrando apenas pedidos delivered sem atraso)
```

**Leitura:** Mesmo eliminando completamente o efeito do atraso logístico, quase 5% das avaliações negativas vêm de problemas de produto/catálogo. São causas independentes que exigem um agente específico de auditoria de anúncio.

---

#### Referência para gráfico — Problema 4: Comunicação e reputação

```
Cobertura de texto nos reviews
────────────────────────────────────────────────────────────────────
Com comentário de texto        41,3%  ████████████
Sem comentário                 58,7%  █████████████████
────────────────────────────────────────────────────────────────────

Notas baixas (1–2) sem nenhum texto explicativo
────────────────────────────────────────────────────────────────────
Com texto     91,9%  ███████████████████████████████████████████
Sem texto      8,1%  ████
────────────────────────────────────────────────────────────────────

Pedidos "entregues" mas contestados pelo cliente
────────────────────────────────────────────────────────────────────
Total de pedidos contestados:  1.718
Nota média desses pedidos:     1,45
────────────────────────────────────────────────────────────────────
Fonte: order_reviews (message_review + review_score + order status)
```

**Leitura:** 1 em cada 12 clientes insatisfeitos não deixa nenhuma pista textual. Os 1.718 pedidos contestados representam um risco de integridade — o sistema registra entrega, o cliente nega. Um agente de classificação de reviews e detecção de anomalias consegue cobrir esses dois gaps simultaneamente.

---

#### Referência para gráfico — Problema 5: Retenção e recompra

```
Taxa de recompra por segmento de cliente
────────────────────────────────────────────────────────────────────
Base geral (todos os clientes)          ██  3,0%
Categorias de reposição/consumo         █████████  18,1%
────────────────────────────────────────────────────────────────────

Correlação nota × probabilidade de recompra
────────────────────────────────────────────────────────────────────
Coeficiente de correlação:  0,006  (estatisticamente nula)
────────────────────────────────────────────────────────────────────

Janela temporal entre 1ª e 2ª compra (clientes que recompraram)
────────────────────────────────────────────────────────────────────
Mediana:  28 dias
Após 60 dias: probabilidade de retorno cai significativamente
────────────────────────────────────────────────────────────────────
                   28d         60d
  ████████████████ │ ████████  │  ██
  (alta propensão) │ (declínio)│  (baixa)
────────────────────────────────────────────────────────────────────
Fonte: orders (customer_unique_id + order_purchase_timestamp)
```

**Leitura:** Melhorar a experiência é necessário, mas insuficiente para reter clientes — a correlação nula entre nota e recompra prova isso. A janela de 28 dias é o sinal mais acionável: um agente de reativação tem uma janela curta e definida para agir com máxima efetividade.

---

### 7.3 O que vem na v3

Com os cinco problemas identificados e suas evidências consolidadas (seções 7.1 e 7.2), a v3 deste documento irá produzir:

- **Mapa de agentes de IA:** para cada um dos cinco problemas, um agente com objetivo, gatilho, entradas, saídas e usuários envolvidos;
- **Arquitetura conceitual:** fontes de dado, fluxo de informação e interações entre agentes (incluindo como o agente de reputação alimenta o agente de desempenho de vendedores);
- **Prompts de referência:** estrutura de prompt para cada agente, orientada pelos padrões de IA Agentic;
- **Critérios de sucesso:** métricas de negócio que cada agente deverá mover, derivadas diretamente das evidências desta seção.
