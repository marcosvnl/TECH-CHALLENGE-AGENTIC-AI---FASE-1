# Relatório Executivo — Olist Intelligent Marketplace
### Tech Challenge — Fase 1 | Fundamentos de IA e Agentic AI
**v1 — Diagnóstico de Negócio** (mapa de agentes, arquitetura conceitual e prompts serão adicionados na v2)

---

## 1. Sumário Executivo

A Olist processou aproximadamente 100 mil pedidos entre 2016 e 2018, conectando pequenos e médios lojistas aos maiores marketplaces do Brasil. A análise integrada das oito bases públicas do dataset (pedidos, itens, pagamentos, avaliações, produtos, vendedores, clientes e geolocalização) revela que os problemas mais graves da operação não estão na aquisição de clientes — a plataforma converte bem — e sim na experiência pós-compra e na retenção. A nota média da plataforma é 4,09, mas a taxa de recompra é de apenas 3%, e um conjunto de causas estruturais e operacionais recorrentes explica a maior parte das avaliações negativas.

Esta análise identificou e quantificou dez frentes de problema, cobrindo cerca de 86% de todas as avaliações nota 1-2 registradas na base. Cada uma dessas frentes está ancorada em evidência de dado (não em suposição), e este documento é a base sobre a qual, na próxima etapa, serão desenhados os agentes de IA responsáveis por endereçá-las.

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

## 5. Síntese de Priorização

Cruzando impacto (magnitude do efeito na nota), abrangência (% de pedidos afetados) e controlabilidade (grau de controle direto da Olist), a leitura executiva é a seguinte. As alavancas de maior retorno por esforço — alto controle, resultado rápido — são o SLA de despacho do vendedor (baixo volume, efeito de 5x na taxa de atraso final) e a correção da política de frete por item em pedidos multi-item (problema de desenho de produto, não de terceiros). As causas de maior abrangência mas menor controle direto são o atraso geral e a dependência da malha de transporte terceirizada, especialmente em picos sazonais e nas regiões Norte/Nordeste. Ruptura de estoque e divergência de catálogo (item errado, quantidade parcial) são problemas de qualidade de operação do vendedor, com controle indireto mas abordável via auditoria e regras de plataforma. E, por fim, a retenção de clientes é uma frente estratégica paralela — não compete com as demais por prioridade, porque ataca uma causa completamente diferente (a ausência de gatilho de recompra, não a qualidade da experiência).

---

## 6. Limitações da Análise

A base cobre o período de 2016 a 2018 e pode não refletir integralmente a operação atual da Olist. A leitura de causa-raiz via texto de review tem cobertura parcial (41,3% das avaliações têm comentário), e 8,1% das notas baixas não têm nenhuma explicação textual disponível. Correlações identificadas (por exemplo, parcelamento longo e nota mais baixa) indicam associação, não necessariamente causalidade direta, e foram reportadas como sinais moderados, não como conclusões definitivas.

---

## 7. Próximos Passos (v2 deste documento)

Este diagnóstico é o insumo direto para a etapa seguinte do projeto: a tradução de cada uma das dez frentes acima em agentes de IA específicos, com objetivo, usuários envolvidos e benefício esperado; o desenho da arquitetura conceitual (fontes de dado, agentes, entradas/saídas, interações entre agentes); e a estruturação de prompts de referência para cada agente proposto. A matriz de priorização da seção 5 vai orientar qual agente é desenhado primeiro — não pela sofisticação técnica, mas pelo retorno esperado sobre o problema mais crítico e mais controlável pela Olist.
