# Análise Exploratória e Diagnóstica da Brew Haven usando LLM

## Problema de Negócio

### Contexto da empresa

A Brew Haven é uma rede de cafeterias que começou com uma pequena loja em um bairro movimentado da cidade. Com o passar dos anos, a empresa expandiu e hoje possui diversas unidades espalhadas pela cidade.

A rede vende diversos produtos, como:

- Cafés tradicionais
- Bebidas geladas
- Chás
- Chocolates
- Produtos de confeitaria

O negócio funciona de forma semelhante a muitas cafeterias modernas:

- Grande fluxo de clientes ao longo do dia
- Vendas rápidas no balcão
- Ticket médio relativamente baixo
- Grande volume de transações

### A dúvida da diretoria

Nos últimos meses, os gestores da Brew Haven começaram a perceber que algumas lojas parecem estar **cada vez mais movimentadas**, principalmente em determinados horários do dia.

Diante desse cenário, a Brew Haven decidiu contratar um Analista de Dados para investigar a seguinte questão:

> **Existe crescimento no faturamento da rede de cafeterias? Se sim, quais são as principais alavancas desse crescimento?**

Esse é o **problema de negócio que vamos investigar**.

---

## Premissas da análise

- Dados de aproximadamente **148.969 transações** de uma rede de cafeterias com **3 lojas em Nova York** (Astoria, Lower Manhattan, Hell's Kitchen).
- Vendas realizadas no período de **Janeiro a Junho de 2023**.
- Cada linha representa um **item dentro de um pedido**, permitindo análises de mix de produtos, ticket médio e comportamento por horário.

---

## Estratégia da solução

O método **Fato-Dimensão** foi usado para desenvolver a análise de dados.

### Passo 1: Resumir o contexto em uma pergunta aberta

As perguntas abertas são um tipo de demanda muito comum em análise de dados nas quais a demanda possui **N possíveis soluções** e cabe ao analista de dados avaliar as possibilidades e escolher a alternativa com o maior retorno e o menor esforço possível.

Para essa análise, foi definida a seguinte pergunta aberta:

> **Como estão as vendas da rede de cafeterias? O faturamento está crescendo e quais são as principais alavancas por trás desse crescimento?**

### Passo 2: Transformar pergunta aberta em fechada

As perguntas fechadas são um tipo de demanda muito comum na área de análise de dados. Essa demanda contém todos os detalhes da análise de dados e direciona o analista exatamente para o que precisa ser feito. Geralmente, a pergunta fechada é a escolha de uma solução entre todas as alternativas possíveis, feita por um profissional mais sênior da área.

Para essa análise, foi definida a seguinte pergunta fechada:

> **Pergunta Fechada:** Analise o crescimento mês a mês do faturamento total e identifique se ele é impulsionado por aumento de volume de clientes ou por aumento do ticket médio. Em seguida, desdobre essa análise por loja, por categoria de produto, por tamanho de bebida, por horário do dia e por dia da semana, para identificar quais fatores estão puxando (ou travando) o crescimento da rede.

### Passo 3: Definição da Coluna Fato

O **Fato** é a coluna de interesse que representa o ponto focal da análise. Nesse caso, a coluna **`transaction_id`** mostra quantas vendas (checkouts) foram feitas, quando, e é feita uma contagem (utilizando valores distintos, já que um mesmo `transaction_id` pode aparecer em várias linhas quando o cliente leva múltiplos produtos) e segmentação por categoria (como loja, mês, dia da semana, tipo de produto e tamanho).

O valor associado a esse fato é o **`Total_Bill`** do pedido, que deve ser somado por `transaction_id` para obter o faturamento real de cada venda.

### Passo 4: Identificação das Dimensões

#### Dimensões Analisadas do dataframe da Brew Haven

**Entendimento da Base de Dados - Visão geral das colunas**

| Coluna | Tipo | O que representa | Significado de negócio |
|--------|------|------------------|------------------------|
| transaction_id | Numérico | Identificador único da transação | Chave primária — cada linha é uma venda de um único item (não é o pedido inteiro, é o item vendido) |
| transaction_date | Data (texto) | Data da venda (DD-MM-AAAA) | Permite análise de sazonalidade, dias da semana, feriados |
| transaction_time | Hora (texto) | Horário exato da venda | Permite entender picos de movimento (manhã, almoço, tarde) |
| store_id | Numérico | Código da loja | Identificador técnico da unidade |
| store_location | Categórico | Nome/bairro da loja (Hell's Kitchen, Lower Manhattan, Astoria) | Permite comparar performance entre unidades |
| product_id | Numérico | Código do produto | Identificador técnico do item vendido |
| transaction_qty | Numérico | Quantidade vendida naquela transação | Volume de vendas — indica se é venda unitária ou em maior quantidade |
| unit_price | Monetário (texto "R$ x,xx") | Preço unitário do produto | Base para entender posicionamento de preço por produto |
| Total_Bill | Monetário (texto "R$ x,xx") | Valor total da linha (qty × unit_price) | Métrica-chave de receita |
| product_category | Categórico | Categoria macro do produto (Coffee, Tea, Bakery, etc.) | Visão de portfólio/mix de produtos |
| product_type | Categórico | Subcategoria (ex: dentro de "Coffee": Espresso, Latte...) | Granularidade intermediária do mix |
| product_detail | Categórico | Nome específico do item | Nível mais detalhado — SKU |
| Size | Categórico | Tamanho (Small, Regular, Large, Not Defined) | Relevante para bebidas; "Not Defined" provavelmente aparece em produtos sem tamanho (ex: grãos de café, itens de padaria) |
| Month Name / Month | Categórico/Numérico | Mês da transação (nome e número) | Sazonalidade mensal |
| Day Name / Day of Week | Categórico/Numérico | Dia da semana (nome e número 0-6) | Padrão semanal (fim de semana vs. dia útil) |
| Hour | Numérico | Hora da transação (0-23) | Padrão intradiário — horários de pico |

**Observação importante:** essa é uma base em **grão de item vendido**, não de "pedido" ou "cliente". Isso significa que não temos como saber quantos itens diferentes um mesmo cliente comprou em uma única visita, nem temos um identificador de cliente. Ou seja, dá para analisar produtos, receita, tempo e loja, mas não dá para analisar comportamento de cliente individual (recorrência, ticket médio por cliente, etc.) — o que temos é ticket médio por transação/linha, não por visita.

#### Quais tipos de análise podem ser feitas com esse dataframe?

- Análise de receita e volume ao longo do tempo — evolução mês a mês, dia a dia, por hora do dia.
- Análise comparativa entre lojas — receita, volume e mix de produtos por unidade (Hell's Kitchen vs. Lower Manhattan vs. Astoria).
- Análise de mix de produtos (portfólio) — quais categorias/tipos/itens mais vendem em quantidade e em receita (ex: análise 80/20).
- Análise de sazonalidade e padrão temporal — dias da semana mais fortes, horários de pico, tendência ao longo dos 5 meses.
- Análise de precificação — variação de preço unitário por produto/tamanho, e como isso se traduz em receita.
- Análise de tamanho/formato preferido (Small/Regular/Large) para itens que têm essa variação.

### Passo 5: Hipóteses Analíticas

#### Bloco 1 — Tendência e Performance Geral

**H1:** A receita está em trajetória de crescimento ao longo dos 5 meses  
- **Lógica:** Se a empresa está saudável, esperamos ver receita mensal estável ou crescente entre janeiro e maio de 2023. Uma queda consistente seria um sinal de alerta relevante para o CEO.  
- **Como testar:** Agrupar `Total_Bill` por `Month`/`Month Name`, somar receita total e volume (`transaction_qty`), e plotar a série mensal. Calcular variação percentual mês a mês (MoM).

**H2:** Existe sazonalidade dentro da semana — fins de semana e/ou dias específicos vendem mais  
- **Lógica:** Cafeterias tipicamente têm padrão de consumo diferente entre dias úteis (rotina de trabalho) e fins de semana (lazer). Entender isso ajuda o CEO a avaliar se a operação está capturando bem esses padrões.  
- **Como testar:** Agrupar receita e volume por `Day Name`/`Day of Week`, comparar médias e visualizar em gráfico de barras.

**H3:** Existem horários de pico concentrados (ex: manhã, horário de almoço)  
- **Lógica:** Cafeterias costumam ter picos claros (café da manhã, pausa da tarde). Isso não é uma decisão operacional imediata, mas ajuda a contextualizar se o padrão de consumo é "saudável"/esperado para o setor.  
- **Como testar:** Agrupar receita/volume por `Hour`, visualizar curva intradiária.

#### Bloco 2 — Comparação entre Lojas

**H4:** As 3 lojas têm performances de receita significativamente diferentes  
- **Lógica:** Se uma loja puxa a média para cima e as outras estão fracas (ou vice-versa), a visão consolidada pode esconder problemas ou oportunidades relevantes para o CEO.  
- **Como testar:** Agrupar `Total_Bill` por `store_location`, comparar receita total, receita média por transação e volume. Calcular participação (%) de cada loja na receita total.

**H5:** As lojas têm trajetórias de crescimento diferentes ao longo do tempo  
- **Lógica:** A empresa como um todo pode parecer estável, mas isso pode mascarar uma loja crescendo e outra em queda — informação crítica para o CEO avaliar "bom ou ruim" com mais nuance.  
- **Como testar:** Cruzar `store_location` x `Month`, gerando série temporal de receita por loja (gráfico de linhas múltiplas).

**H6:** O mix de produtos varia entre lojas (cada loja tem um "perfil" de consumo diferente)  
- **Lógica:** Localização (Hell's Kitchen, Lower Manhattan, Astoria) tem perfis de público diferentes (turismo, área comercial, residencial), o que pode gerar diferenças no que é mais vendido.  
- **Como testar:** Tabela cruzada `store_location` x `product_category`, comparando % de receita por categoria dentro de cada loja.

#### Bloco 3 — Mix de Produtos e Portfólio

**H7:** Uma pequena quantidade de categorias/produtos concentra a maior parte da receita (padrão 80/20)  
- **Lógica:** É comum que poucos produtos "carreguem" o negócio. Saber isso ajuda o CEO a entender onde está a real força do portfólio.  
- **Como testar:** Agrupar receita por `product_category` e `product_type`, ordenar decrescente, calcular % acumulado (curva de Pareto).

**H8:** Itens de tamanho "Large" geram receita desproporcional em relação ao volume vendido  
- **Lógica:** Se tamanhos maiores têm preço proporcionalmente mais alto que o custo/volume adicional, isso indica eficiência de ticket — relevante mesmo numa análise geral, pois mostra "qualidade" da receita.  
- **Como testar:** Agrupar `Total_Bill` e `transaction_qty` por `Size` (excluindo "Not Defined"), comparar ticket médio por tamanho.

**H9:** "Coffee beans" e itens de maior preço unitário têm baixo volume mas impacto relevante na receita  
- **Lógica:** Categorias como grãos premium (ex: "Civet Cat" visto na amostra) têm preço unitário alto — mesmo com poucas transações, podem distorcer a leitura de "ticket médio" e merecem ser vistas separadamente.  
- **Como testar:** Comparar distribuição de `unit_price` por categoria, identificar outliers de preço e seu peso relativo na receita total.

#### Bloco 4 — Qualidade e Consistência da Receita

**H10:** O ticket médio por transação é estável ao longo do tempo (sinal de consistência, não só de volume)  
- **Lógica:** Receita pode crescer por mais transações (mais fluxo) ou por ticket maior (mais valor por venda). Isso muda a "leitura" de saúde do negócio para o CEO — crescer por volume é diferente de crescer por preço.  
- **Como testar:** Calcular receita total, número de transações e ticket médio (`Total_Bill`/transação) por mês, decompor a variação de receita em efeito volume vs. efeito ticket médio.

---

### Passo 6: Critérios de Priorização

- **Critério 1:** Dados disponíveis.
- **Critério 2:** Insights acionáveis.

### Passo 7: Priorização das Hipóteses Analíticas

**Resumo dos vereditos**

| # | Hipótese | Veredito |
|---|----------|----------|
| H1 | Receita em trajetória de crescimento | ✅ Confirmada |
| H2 | Fim de semana vende mais | ❌ Refutada |
| H3 | Picos de horário concentrados | ✅ Confirmada |
| H4 | Lojas com performance muito diferente | ❌ Refutada |
| H5 | Lojas com trajetórias diferentes | ❌ Refutada |
| H6 | Mix de produto varia por loja | 🟡 Parcial |
| H7 | Concentração de receita (Pareto) | ✅ Confirmada |
| H8 | Tamanho maior = ticket desproporcional | ✅ Confirmada |
| H9 | Itens caros, baixo volume, alto impacto | 🟡 Parcial |
| H10 | Ticket médio estável (crescimento = volume) | ✅ Confirmada |

---

## Insights da análise

### Resultados

**📥 Baixe a apresentação em HTML estilo PowerPoint** (clique no link e, em seguida, em "Download" ou "View raw"):  
[https://drive.google.com/file/d/1E27NQAoMS-IlWDIjmUC06cNWLhDytVfQ/view?usp=sharing](https://drive.google.com/file/d/1E27NQAoMS-IlWDIjmUC06cNWLhDytVfQ/view?usp=sharing)

**📥 Baixe a apresentação em HTML estilo Power BI** (clique no link e, em seguida, em "Download" ou "View raw"):  
[https://drive.google.com/file/d/1DpqMkLNrnWYtOHN_XdZFHJoG2ZFfss67/view?usp=sharing](https://drive.google.com/file/d/1DpqMkLNrnWYtOHN_XdZFHJoG2ZFfss67/view?usp=sharing)

---

## Próximos passos

Receber feedback do CEO para saber se seu desejo foi atendido com base na análise aqui presente, ou se será feito um aprofundamento na análise.
