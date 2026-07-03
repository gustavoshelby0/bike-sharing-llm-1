# Análise Exploratória e Diagnóstica da ?? usando LLM

## Problema de Negócio

### Contexto da empresa

Com base nas suas respostas, aqui está o contexto de negócio consolidado:

**Quem consome:** CEO da empresa de bike-sharing.

**Dor / pergunta central:** A empresa performou bem ou mal nesse período? O setor de bike-sharing como um todo está em expansão ou estagnação?

**Decisão que a análise embasa:** Decisão de **investir mais** ou **descontinuar/cortar** essa linha de produto/negócio.

**Implicações para o desenho da análise** (para eu alinhar com você antes de seguir):

- Como a decisão é binária e de alto risco (investir vs. cortar), a análise precisa ir além de "o volume subiu" — precisa mostrar **trajetória de crescimento** (mês a mês, YoY), **qualidade desse crescimento** (usuários recorrentes vs. ocasionais) e **sensibilidade a fatores externos** (clima, sazonalidade, feriados) que ajudam a separar "crescimento estrutural do negócio" de "ruído/sazonalidade".
- Uma limitação importante: essa base é **operacional** (viagens/hora), não tem receita, custo, churn de assinantes ou CAC. Então dá para responder bem "a demanda está crescendo?", mas uma resposta completa sobre "investir ou cortar" idealmente cruzaria isso com dados financeiros, se existirem. Vale eu sinalizar isso nas recomendações finais.

## Premissas da análise

A base tem 10.886 registros horários, cobrindo 01/01/2011 a 19/12/2012 (~2 anos), sem valores nulos — uma base limpa e granular, ideal para análise de séries temporais e padrões de demanda.
---

## Estratégia da solução

O método **Fato-Dimensão** foi usado para desenvolver a análise de dados.

### Passo 1: Resumir o contexto em uma pergunta aberta

As perguntas abertas são um tipo de demanda muito comum em análise de dados nas quais a demanda possui **N possíveis soluções** e cabe ao analista de dados avaliar as possibilidades e escolher a alternativa com o maior retorno e o menor esforço possível.

Para essa análise, foi definida a seguinte pergunta aberta:

> **Como esta o crescimento da categoria de bike-sharing? devemos cortar esse setor? ou investir?**

### Passo 2: Transformar pergunta aberta em fechada

As perguntas fechadas são um tipo de demanda muito comum na área de análise de dados. Essa demanda contém todos os detalhes da análise de dados e direciona o analista exatamente para o que precisa ser feito. Geralmente, a pergunta fechada é a escolha de uma solução entre todas as alternativas possíveis, feita por um profissional mais sênior da área.

Para essa análise, foi definida a seguinte pergunta fechada:

> **Pergunta Fechada:** quero um grafico de barra mostrando a evolucao da demanda de bike-sharing durante o tempo, para entender se e algo meramente sazional ou se a categoria esta indo pra frente

### Passo 3: Definição da Coluna Fato

O **Fato** é a coluna de interesse que representa o ponto focal da análise. (dataframe)

datetime — carimbo horário do registro. É a espinha dorsal da análise: permite decompor em ano, mês, dia da semana e hora, essencial para separar tendência (crescimento estrutural) de sazonalidade (padrão que se repete).

### Passo 4: Identificação das Dimensões

#### Dimensões Analisadas do dataframe

#### O que cada coluna representa (e seu significado de negócio)

**Dimensões de tempo/contexto**

- **`datetime`** — carimbo horário do registro. É a espinha dorsal da análise: permite decompor em ano, mês, dia da semana e hora, essencial para separar tendência (crescimento estrutural) de sazonalidade (padrão que se repete).
- **`season`** (1=inverno, 2=primavera, 3=verão, 4=outono) — estação do ano. Negócio: bike-sharing é fortemente sazonal; ajuda a entender se quedas de volume são "o negócio piorando" ou "só é inverno".
- **`holiday`** (0/1) — se o dia é feriado. Negócio: muda o perfil de uso (mais lazer, menos deslocamento para trabalho).
- **`workingday`** (0/1) — se é dia útil (não é fim de semana nem feriado). Negócio: separa uso "utilitário" (ir trabalhar) de uso "recreativo" (passeio).

**Dimensões climáticas**

- **`weather`** (1=limpo → 4=chuva/neve severa) — condição do tempo. Negócio: clima ruim reduz demanda; é um fator externo, não controlável pela empresa, importante para não confundir "queda por clima" com "queda por perda de mercado".
- **`temp`** / **`atemp`** — temperatura real e sensação térmica (°C). Negócio: existe uma faixa de conforto que maximiza aluguéis; temperaturas extremas (muito frio ou muito quente) derrubam a demanda.
- **`humidity`** — umidade relativa do ar. Negócio: afeta conforto do ciclista, especialmente combinada com temperatura alta.
- **`windspeed`** — velocidade do vento. Negócio: vento forte desestimula o uso de bicicleta.

**Métricas de resultado (as mais importantes para o CEO)**

- **`casual`** — aluguéis feitos por usuários sem cadastro (avulsos/turistas/ocasionais). Negócio: proxy de **aquisição/demanda esporádica**, mais sensível a clima, fim de semana e lazer.
- **`registered`** — aluguéis feitos por usuários cadastrados/assinantes. Negócio: proxy de **uso recorrente/fidelizado**, geralmente ligado a deslocamento (commuting) em dias úteis — é o motor mais estável e previsível do negócio.
- **`count`** — total de aluguéis na hora (`casual` + `registered`). Negócio: é a métrica-headline de volume/demanda, mas sozinha esconde a composição entre os dois perfis de usuário, que é justamente o que diferencia "crescimento saudável" de "pico passageiro".

---

#### Que tipos de análise podem ser feitas com essa base

1. **Análise de tendência temporal (crescimento/queda)** — evolução mensal e YoY (2011 vs. 2012) de `count`, `casual` e `registered`, para responder diretamente "a empresa/o setor está crescendo?".
2. **Decomposição de sazonalidade** — padrões por estação, mês, dia da semana e hora do dia (ex.: picos às 8h e 18h em dias úteis = padrão de commuting).
3. **Análise de composição de usuários (mix casual vs. registered)** — como a proporção entre os dois evolui ao longo do tempo; sinaliza se o crescimento vem de fidelização (bom sinal estrutural) ou só de picos ocasionais (menos previsível).
4. **Análise de sensibilidade a fatores externos (clima)** — correlação de `count`/`casual`/`registered` com `temp`, `humidity`, `windspeed` e `weather`, para isolar o efeito do clima do efeito de tendência real do negócio.
5. **Segmentação por tipo de dia** — comparação `workingday` vs. fim de semana/feriado, e o que isso revela sobre o "job to be done" do produto (transporte utilitário vs. lazer).
6. **Detecção de anomalias/outliers** — horas com volume muito acima/abaixo do esperado, que podem indicar eventos especiais, falhas ou oportunidades.

#### Perguntas de negócio que essa base pode responder

- A demanda total (`count`) está crescendo, estagnada ou caindo entre 2011 e 2012?
- O crescimento é impulsionado por novos usuários ocasionais (`casual`) ou por uso recorrente de assinantes (`registered`)? Qual é mais saudável/sustentável?
- Existe sazonalidade forte o suficiente para explicar quedas pontuais sem que isso signifique problema estrutural?
- O uso é majoritariamente utilitário (commuting em dias úteis) ou recreativo (fins de semana/feriados)? Isso indica quem é o cliente núcleo do negócio.
- O clima é um fator de risco relevante para a operação (ex.: dependência de dias bons)?
- Existem horários/dias com capacidade ociosa ou sobre-demanda, relevantes para dimensionamento de frota?

**O que essa base NÃO responde diretamente** (importante para o CEO): não há receita, custo, ticket médio, churn de assinantes ou CAC — então ela mede muito bem a **demanda/uso do produto**, mas a decisão final de investir vs. cortar dependeria de cruzar isso com dados financeiros, se disponíveis.


### Passo 5: Hipóteses Analíticas
#### Bloco A — Crescimento do negócio

**H1: A demanda total (`count`) cresceu de 2011 para 2012.**

- **Lógica:** é a pergunta mais direta do CEO — "o negócio está crescendo?". Se sim, é um argumento a favor de investir; se está estagnado ou caindo, é sinal de alerta.
- **Como testar:** agregar `count` por mês e por ano, comparar totais/médias 2011 vs. 2012 (YoY %). Ideal comparar meses equivalentes (jan/2011 vs jan/2012) para não misturar sazonalidade com tendência.

**H2: O crescimento (se existir) é consistente ao longo dos meses, não concentrado em poucos picos.**

- **Lógica:** um crescimento "saudável" é distribuído; se só alguns meses puxam a média, pode ser evento pontual (ex.: campanha, evento na cidade) e não crescimento estrutural do negócio.
- **Como testar:** plotar a série mensal de `count` para os 2 anos sobrepostos (mês a mês) e observar se o ganho é generalizado ou concentrado.

---

#### Bloco B — Composição de usuários (casual vs. registered)

**H3: O crescimento é puxado principalmente por usuários `registered`, não por `casual`.**

- **Lógica:** usuários registrados representam uso recorrente/fidelizado — um motor de crescimento mais previsível e sustentável do que picos de usuários ocasionais. Isso muda a leitura de "quão saudável" é o crescimento.
- **Como testar:** calcular a taxa de crescimento YoY separadamente para `casual` e `registered`, e comparar a participação (%) de cada um no total ao longo do tempo.

**H4: A proporção de usuários `registered` no total aumenta ao longo do tempo (maior fidelização).**

- **Lógica:** se a empresa está retendo mais usuários (convertendo casual → registrado), isso é evidência de maturação e fortalecimento do negócio, não só de mais volume.
- **Como testar:** calcular `registered / count` por mês e verificar se há tendência de alta ao longo dos 24 meses.

---

#### Bloco C — Sazonalidade e clima

**H5: A demanda é fortemente sazonal (maior no verão/primavera, menor no inverno).**

- **Lógica:** se a sazonalidade explica boa parte da variação, quedas pontuais não devem ser interpretadas como "o negócio piorando" — importante para o CEO não tirar conclusões precipitadas olhando só um trimestre ruim.
- **Como testar:** comparar médias de `count` por `season`, com boxplots ou médias por estação; testar se a diferença é estatisticamente relevante (ex.: ANOVA ou apenas magnitude da diferença de médias).

**H6: Condições climáticas ruins (`weather` alto, chuva/neve) reduzem a demanda, especialmente de usuários `casual`.**

- **Lógica:** usuários ocasionais (lazer/turismo) devem ser mais sensíveis ao clima do que usuários registrados que usam para deslocamento obrigatório (trabalho). Isso testa se `casual` é um segmento mais "frágil" para o negócio.
- **Como testar:** correlacionar `weather`, `temp`, `humidity`, `windspeed` com `casual` e `registered` separadamente, comparando a força da correlação entre os dois grupos.

**H7: Existe uma "faixa de conforto" de temperatura que maximiza os aluguéis, com queda em extremos (muito frio ou muito quente).**

- **Lógica:** relação não-linear — relevante para entender limites operacionais e picos de demanda esperados por clima.
- **Como testar:** plotar `count` médio por faixas (bins) de `temp`/`atemp` e observar se a curva sobe e depois desce (formato de "sino").

---

#### Bloco D — Padrão de uso (utilitário vs. recreativo)

**H8: Em dias úteis (`workingday`=1), o uso é dominado por `registered`, com picos nítidos às 8h e 18h (commuting).**

- **Lógica:** se confirmado, mostra que o core do negócio é transporte utilitário (mais previsível, ligado a rotina de trabalho) — um argumento forte de "produto essencial" para o CEO.
- **Como testar:** agrupar `count`/`registered`/`casual` por hora do dia, separando `workingday`=1 vs. 0, e visualizar os picos horários.

**H9: Em fins de semana e feriados, o uso é mais distribuído ao longo do dia e dominado por `casual` (lazer).**

- **Lógica:** complementar à H8 — caracteriza o segundo "job to be done" do produto (recreação), que tem dinâmica de demanda diferente e pode exigir estratégia própria.
- **Como testar:** mesma agregação por hora, mas para `workingday`=0 e `holiday`=1, comparando o formato da curva com a de dias úteis.

**H10: Feriados têm padrão de uso mais parecido com fins de semana do que com dias úteis comuns.**

- **Lógica:** valida se `holiday` deve ser tratado como uma variante de "dia não-útil" na análise, simplificando a segmentação.
- **Como testar:** comparar curvas horárias médias de `holiday`=1 vs. `workingday`=0 (sem feriado) vs. `workingday`=1.

---

### Passo 6: Critérios de Priorização

- **Critério 1:** Dados disponíveis.
- **Critério 2:** Insights acionáveis.

### Passo 7: Priorização das Hipóteses Analíticas

### Resumo dos vereditos

| # | Hipótese | Veredito |
| --- | --- | --- |
| H1 | Demanda total cresceu 2011→2012 | ✅ Confirmada (+66,7%) |
| H2 | Crescimento consistente em todos os meses | ✅ Confirmada |
| H3 | Crescimento puxado por `registered` | ✅ Confirmada (+70% vs +52%) |
| H4 | Participação de `registered` cresce no tempo | 🟡 Parcial (leve alta, ofuscada por sazonalidade) |
| H5 | Forte sazonalidade por estação | ✅ Confirmada |
| H6 | Clima ruim afeta mais `casual` | ✅ Confirmada |
| H7 | Curva em sino de temperatura (conforto) | ❌ Refutada (curva sobe e estabiliza, não cai) |
| H8 | Dias úteis: picos 8h/18h, dominado registered | ✅ Confirmada |
| H9 | Fim de semana: distribuído, dominado por casual | 🟡 Parcial (distribuído sim; dominado não — registered ainda é maioria) |
| H10 | Feriado parecido com fim de semana | ✅ Confirmada |
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
