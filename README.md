# Teste Prático - Analista de Dados & Automação | Clint

## 1. Auditoria e Qualidade dos Dados (Etapa 1)
Antes de iniciar as análises de negócio, foi realizada uma auditoria técnica para garantir a integridade do banco de dados analítico.

### 1.1 Inconsistências Identificadas
Durante o processo de exploração e limpeza via SQL, foram detectados os seguintes pontos:

#### 1.1.1 Tratamento de Leads Duplicados
Identificação e exclusão lógica de registros marcados com a tag "DUPLICADO" na tabela de contatos para evitar a inflação das métricas de topo de funil.

```sql
-- Consulta para validar volume de duplicatas antes da análise
SELECT COUNT(*) AS total_duplicados
FROM contacts 
WHERE tags LIKE '%DUPLICADO%';

-- Criação de uma View para garantir que as próximas etapas usem apenas dados únicos
CREATE VIEW vw_contacts_unicos AS
SELECT * FROM contacts 
WHERE tags NOT LIKE '%DUPLICADO%' OR tags IS NULL;
```

#### 1.1.2 Saneamento de Chaves Órfãs
Remoção de eventos de movimentação no funil que não possuem um negócio (deal_id) correspondente na tabela principal, garantindo a integridade referencial.

```sql
-- Identificação de registros órfãos na pipeline_events (Retornou 18 linhas)
SELECT * FROM pipeline_events 
WHERE deal_id NOT IN (SELECT deal_id FROM deals);

-- Remoção dos registros sem correspondência para não afetar as taxas de conversão
DELETE FROM pipeline_events 
WHERE deal_id NOT IN (SELECT deal_id FROM deals);
```

#### 1.1.3. Verificação de Consistência Temporal
Análise da cronologia entre a criação do contato e a criação do negócio. Foi identificada uma divergência em aproximadamente 50% da base, onde o negócio consta como criado antes do contato.

```sql
SELECT d.deal_id, d.deal_created_at, c.contact_created_at
FROM deals d
JOIN contacts c ON d.contact_id = c.contact_id
WHERE d.deal_created_at < c.contact_created_at;
```

Decisão técnica: Os registros serão mantidos para preservar o volume histórico, considerando uma possível migração de dados ou registro retroativo no CRM.

#### 1.1.4. Análise de Valores Inconsistentes
Identifiquei uma falha crítica na integridade dos dados de receita, onde negócios marcados como "Ganho" (won) apresentam valor zerado. Foi necessário validar a string exata para evitar falsos positivos com valores que possuem centavos (ex: R$ 10.000,00).

```sql
-- Identificação precisa de vendas ganhas sem valor real (R$ 0,00)
-- Retornou 3 linhas de inconsistência total
SELECT * FROM deals 
WHERE status = 'won' AND valor = 'R$ 0,00';
```
Decisão Técnica: Estes registros serão sinalizados como "Vendas com Erro de Preenchimento". Para cálculos de Ticket Médio e Receita Total, esses valores zerados serão desconsiderados para não distorcer a realidade financeira da operação.

### 1.2 Stack Tecnológica Utilizada
* **Banco de Dados:** SQLite para processamento local.
* **Ferramenta de Gestão:** DB Browser.
* **Linguagem:** SQL para transformação e limpeza.

## 2. Reconstrução do funil de vendas (Etapa 2)

### 2.1 Volume Total de Leads (Únicos) = 815

```sql
SELECT COUNT(*) AS total_leads 
FROM vw_contacts_unicos;
```
### 2.2 Volume de Deals Criados = 632

```sql
SELECT COUNT(*) AS total_deals 
FROM deals;
```

### 2.3 Volume por Etapa do Funil (Pipeline Real)

Esta query lista quantos negócios passaram por cada estágio definido.

| Etapa do Funil | Volume |
|----------------|--------|
|Reunião Realizada|	463|
|Reunião Agendada|	461|
|MQL|	437|
|Proposta	|419|

```sql
SELECT 
    stage, 
    COUNT(DISTINCT deal_id) AS volume_negocios
FROM pipeline_events
WHERE stage IN ('MQL', 'Reunião Agendada', 'Reunião Realizada', 'Proposta')
GROUP BY stage
ORDER BY volume_negocios DESC;
```

### 2.4 Volume de Vendas Realizadas (Validadas) = 121

Filtrei as 3 vendas com valor zerado identificadas na auditoria.

```sql
SELECT COUNT(*) AS vendas_realizadas
FROM deals
WHERE status = 'won' AND valor != 'R$ 0,00';
```
### 2.5 Taxa de conversão entre etapas

Abaixo, a tabela consolidada com os indicadores de eficiência do funil da Clint:

| Etapa | Volume | Conversão por etapa | Conversão Acumulada |
|-------|--------|---------------------|---------------------| 
|Lead   |815     |-                   |100.0%               |
|Deal Criado|632 |77.55%               |77.55%               |
|MQL        |437 |69.15%               |53.62%               |
|Reunião Agendada|461|105.49%          |56.56%               |
|Reunião Realizada|463|100.43%         |56.81%               |
|Proposta|419|90.5%|51.41%                                   |
|Venda(Won)|121|28.88%|14.85%                                |

Lógica utilizada para a obtenção das respostas.

```sql
WITH funnel_base AS (
    SELECT '1. Lead' AS etapa, COUNT(*) AS volume, 1 AS ordem FROM vw_contacts_unicos
    UNION ALL
    SELECT '2. Deal Criado', COUNT(*), 2 FROM deals
    UNION ALL
    SELECT '3. MQL', COUNT(DISTINCT deal_id), 3 FROM pipeline_events WHERE stage = 'MQL'
    UNION ALL
    SELECT '4. Reunião Agendada', COUNT(DISTINCT deal_id), 4 FROM pipeline_events WHERE stage = 'Reunião Agendada'
    UNION ALL
    SELECT '5. Reunião Realizada', COUNT(DISTINCT deal_id), 5 FROM pipeline_events WHERE stage = 'Reunião Realizada'
    UNION ALL
    SELECT '6. Proposta', COUNT(DISTINCT deal_id), 6 FROM pipeline_events WHERE stage = 'Proposta'
    UNION ALL
    SELECT '7. Venda (Won)', COUNT(*), 7 FROM deals WHERE status = 'won' AND valor != 'R$ 0,00'
)
SELECT 
    etapa, 
    volume,
    ROUND(CAST(volume AS REAL) / LAG(volume) OVER (ORDER BY ordem) * 100, 2) || '%' AS conversao_etapa,
    ROUND(CAST(volume AS REAL) / (SELECT volume FROM funnel_base WHERE ordem = 1) * 100, 2) || '%' AS conversao_acumulada
FROM funnel_base
ORDER BY ordem;
```

As taxas superiores a 100% nas etapas de Reunião indicam que negócios estão pulando o status de MQL ou sendo registrados retroativamente no CRM, um ponto de melhoria operacional identificado na análise.

### 2.6 Identificação do Gargalo

O principal gargalo do funil encontra-se na transição de Proposta para Venda (28.88%). Embora a empresa seja eficiente em agendar e realizar reuniões, a perda de 71% dos leads na última milha do funil é o ponto de maior impacto na receita.

## 3. Análise de Receita (Etapa 3)

Nesta etapa, cruzei os dados de conversão com os valores financeiros para entender o ROI (Retorno sobre Investimento) por canal e segmento.

### 3.1 Receita total = R$ 2.143.280,00 e o Ticket Médio = R$ 17.713,06
Filtrei vendas = 'won' e maiores que R$ 0,00 (conforme auditoria anterior dos dados)

```sql
WITH deals_financeiro AS (
    SELECT 
        CAST(REPLACE(REPLACE(REPLACE(valor, 'R$ ', ''), '.', ''), ',', '.') AS REAL) AS valor_num
    FROM deals
    WHERE status = 'won' AND valor != 'R$ 0,00'
)
SELECT 
    SUM(valor_num) AS receita_total,
    AVG(valor_num) AS ticket_medio
FROM deals_financeiro;
```

### 3.2 Receita por Origem de Marketing

| Origem | Receita Total | Volume de Vendas |
|--------|---------------|------------------|
|Meta Ads|	R$ 403.664	|26|
|Parcerias|	R$ 340.028	|17|
|Orgânico|	R$ 306.210	|15|
|Google Ads|	R$ 296.305	|18|
|Indicação|	R$ 284.766|	16|
|Outbound|	R$ 254.665|	13|
|Eventos|	R$ 224.241|	14|

Código SQL utilizado:

```sql
SELECT 
    c.origem,
    SUM(CAST(REPLACE(REPLACE(REPLACE(d.valor, 'R$ ', ''), '.', ''), ',', '.') AS REAL)) AS receita_total,
    COUNT(d.deal_id) AS volume_vendas
FROM deals d
JOIN vw_contacts_unicos c ON d.contact_id = c.contact_id
WHERE d.status = 'won' AND d.valor != 'R$ 0,00'
GROUP BY c.origem
ORDER BY receita_total DESC;
```
Durante a análise da Etapa 3, identifiquei que a soma da receita por Origem e Segmento (R$ 2.109.879,00) apresenta uma diferença de R$ 33.401,00 em relação à Receita Total (R$ 2.143.280,00).

```sql
SELECT 
    d.deal_id, 
    d.contact_id, 
    d.valor, 
    c.segmento,
    c.origem,
    -- Identificando o motivo da falha na atribuição
    CASE 
        WHEN c.contact_id IS NULL THEN 'Contato filtrado ou inexistente'
        WHEN c.segmento IS NULL OR c.segmento = '' THEN 'Segmento não preenchido'
        ELSE 'Outro erro de integridade'
    END AS motivo_discrepancia
FROM deals d
LEFT JOIN vw_contacts_unicos c ON d.contact_id = c.contact_id
WHERE d.status = 'won' 
  AND d.valor != 'R$ 0,00'
  AND (c.contact_id IS NULL OR c.segmento IS NULL OR c.segmento = '');
```

| deal_id | contact_id | Valor | Segmento | Origem |Motivo|
|---------|------------|-------|----------|--------|------|
|d_00250|	c_00809	|R$ 25.074|	NULL|NULL|Contato filtrado ou inexistente|
|d_00338|	c_00811	|R$ 8.327|	NULL|NULL|Contato filtrado ou inexistente|


A discrepância de 1,55% é considerada baixa para fins de análise de tendência, porém crítica para fins de conciliação financeira.

Recomendação Técnica: Implementar uma trava de sistema (ou automação via API) que impeça a alteração do status de um negócio para "Won" caso o contacto associado não possua os campos de Origem e Segmento preenchidos. Isso garantirá 100% de rastreabilidade do ROI de marketing e performance por setor.

### 3.3 Receita por Segmento

| Segmento | Receita Total | Ticket Médio |
|----------|---------------|--------------|
|Educação|	R$ 410.723|	R$ 19.558|
|Advocacia|	R$ 360.518|	R$ 18.025|
|Saúde|	R$ 299.775|	R$ 21.412|
|Serviços|	R$ 288.564|	R$ 19.237|
|Agência|	R$ 271.857|	R$ 19.418|
|Infoprodutor|	R$ 194.011|	R$ 12.934|
|Indústria|	R$ 166.907|	R$ 16.690|
|E-commerce|	R$ 117.524|	R$ 11.752|

Código SQL utilizado:

```sql
SELECT 
    c.segmento,
    SUM(CAST(REPLACE(REPLACE(REPLACE(d.valor, 'R$ ', ''), '.', ''), ',', '.') AS REAL)) AS receita_total,
    AVG(CAST(REPLACE(REPLACE(REPLACE(d.valor, 'R$ ', ''), '.', ''), ',', '.') AS REAL)) AS ticket_medio_segmento
FROM deals d
JOIN vw_contacts_unicos c ON d.contact_id = c.contact_id
WHERE d.status = 'won' AND d.valor != 'R$ 0,00'
GROUP BY c.segmento
ORDER BY receita_total DESC;
```

### 3.4 Tempo Médio de Ciclo de Vendas = 67 Dias

Como 'Venda' não é um estágio na pipeline_events, usamos a última movimentação do deal para calcular o tempo decorrido desde a criação.

```sql
WITH data_fechamento AS (
    SELECT deal_id, MAX(moved_at) AS ultima_movimentacao
    FROM pipeline_events
    GROUP BY deal_id
)
SELECT 
    AVG(julianday(df.ultima_movimentacao) - julianday(d.deal_created_at)) AS ciclo_medio_dias
FROM deals d
JOIN data_fechamento df ON d.deal_id = df.deal_id
WHERE d.status = 'won' AND d.valor != 'R$ 0,00';
```

### 3.5 Performance dos Closers (Receita Por Vendedor)

| Closer | Receita Fechada | Total de Vendas |
|----------|---------------|--------------|
|Camile Silveira|	R$ 520.070|	28|
|Gustavo|	R$ 404.889|	22|
|Norton	|R$ 359.838|	21|
|Rafael Grizza|	R$ 346.299|	21|
|Beatriz	|R$ 267.052|	18|
|Luis|	R$ 245.132|	11|

Código SQL utilizado:

```sql
SELECT 
    closer,
    SUM(CAST(REPLACE(REPLACE(REPLACE(valor, 'R$ ', ''), '.', ''), ',', '.') AS REAL)) AS receita_fechada,
    COUNT(*) AS total_vendas
FROM deals
WHERE status = 'won' AND valor != 'R$ 0,00'
GROUP BY closer
ORDER BY receita_fechada DESC;
```

## 4. Visualização e Business Intelligence (Etapa 4)

Para a construção do dashboard executivo e validação visual das métricas, os dados brutos foram importados para o Power BI. Com o objetivo de garantir a integridade dos dados e a performance do relatório, foi aplicada uma arquitetura robusta dividida em três pilares: ETL, Modelagem Relacional e Expressões DAX.

### 4.1. Extração, Transformação e Carga (ETL no Power Query)

Toda a lógica de saneamento validada na Etapa 1 (SQL) foi replicada no Power Query utilizando a linguagem M. As tabelas foram segregadas em Dimensões e Fatos:

* d_contacts (Dimensão de Contatos):

    * Filtro aplicado na coluna tags removendo registros que continham "DUPLICADO".

    * Remoção de duplicatas baseada na chave contact_id, garantindo a unicidade da tabela dimensão.

* f_pipeline (Fato de Eventos de Funil):

    * Realizado um Merge (Mescla de Consultas - Inner Join) com a tabela de negócios para remover as 18 chaves órfãs (eventos sem negócios atrelados).

* f_deals (Fato de Negócios):

    * Limpeza da string da coluna valor (remoção do prefixo "R$", substituição de pontos e vírgulas) e conversão para o tipo numérico Decimal/Moeda.

    * Remoção de duplicatas na coluna deal_id (erro identificado na base bruta durante a modelagem), assegurando a integridade para os relacionamentos subsequentes.

### 4.2. Modelagem de Dados (Star Schema)

O modelo foi estruturado no formato Estrela (Star Schema), garantindo que os filtros fluam corretamente de cima para baixo (das dimensões para as fatos).

| Tabela Origem (Lado 1) | Tabela Destino (Lado *) | Chave de Relacionamento | Cardinalidade |
|------------------------|-------------------------|-------------------------|---------------|
|d_contacts|	f_deals|	contact_id| 1:N| 
|d_contacts|	f_marketing|	contact_id| 1:N|
|f_deals|	f_pipeline|	deal_id|1:N |
|f_deals|	f_activities|	deal_id|1:N|

A modelagem em Star Schema validou a inconsistência descoberta na Etapa 3. Embora existam 121 vendas brutas marcadas como "Won", 2 destas vendas estavam atreladas a IDs de contatos marcados como "DUPLICADO".

Como a dimensão d_contacts foi higienizada, a integridade referencial do Power BI filtrou automaticamente esses 2 negócios órfãos, ajustando o volume de Vendas Validadas Líquidas para 119 (Receita de R$ 2.109.879,00). O dashboard foi desenvolvido sobre esta base higienizada para refletir o ROI real.

<img width="637" height="451" alt="image" src="https://github.com/user-attachments/assets/546dd8f2-2815-4dc5-95ac-10867ece387c" />

### 4.3. Regras de Negócio e Medidas (DAX)

Para o cálculo dinâmico das métricas no Dashboard, foram criadas medidas explícitas em DAX, respeitando as regras de validação

1. Total de Leads

```dax
Total Leads = COUNTROWS('d_contacts')
```

2. Volume de Vendas

```dax
Total Vendas (Won) = 
CALCULATE(
    DISTINCTCOUNT('f_deals'[deal_id]),
    'f_deals'[status] = "won",
    'f_deals'[valor] > 0
)
```

3. Receita Total

```dax
Receita Total = 
CALCULATE(
    SUM('f_deals'[valor]),
    'f_deals'[status] = "won",
    'f_deals'[valor] > 0
)
```

4. Ticket Médio Global

```dax
Ticket Medio = DIVIDE([Receita Total], [Total Vendas (Won)], 0)
```
### 4.4 Protótipo Web e Escolha dos KPIs 

Como complemento à solução de BI, desenvolvi um protótipo de Dashboard Executivo Web utilizando HTML, Tailwind CSS e JavaScript. O objetivo desta entrega é demonstrar proficiência em Data App Development e componentização visual com filtros dinâmicos que simulam o motor relacional.

Para a visão executiva superior (Cards Principais), foram selecionados os 4 KPIs mais críticos para a operação da Clint:

    * Total de Leads Únicos: Mede o esforço de atração e o tamanho real do topo de funil (excluindo cadastros duplicados).

    * Vendas Validadas: Indica a eficiência de fechamento (fundo de funil), filtrando negócios perdidos ou com valores zerados inconsistentes.

    * Receita Total: O principal indicador de sucesso comercial (ROI).

    * Ticket Médio: Termômetro da qualidade das vendas e do perfil do cliente (Receita / Vendas).

Código HTML para a construção dos cards principais:

```html
Cards_principais = 
VAR vLeads = FORMAT([Total Leads], "#,0")
VAR vVendas = FORMAT([Total Vendas (Won)], "#,0")
VAR vReceita = FORMAT([Receita Total], "R$ #,0.00")
VAR vTicket = FORMAT([Ticket Medio], "R$ #,0.00")

VAR htmlString = "
<!DOCTYPE html>
<html>
<head>
<style>
  /* Reset e estrutura principal */
  .clint-cards-wrapper { 
      display: flex; gap: 20px; font-family: 'Segoe UI', sans-serif; 
      width: 100%; flex-wrap: wrap; padding: 10px; box-sizing: border-box;
  }
  
  /* Estilo do Cartão Escuro com Tooltip Trigger */
  .clint-card { 
      flex: 1; min-width: 200px; background: #110c1f; border-radius: 12px; 
      padding: 24px; position: relative; overflow: visible; 
      box-shadow: 0 8px 20px rgba(0,0,0,0.4); border: 1px solid #2a1f43;
      transition: transform 0.2s ease, border-color 0.2s ease;
      cursor: help; /* Cursor de interrogação/ajuda */
  }
  
  .clint-card:hover { transform: translateY(-2px); border-color: #db2777; }

  /* Linha de Degradê Superior */
  .clint-card::before { 
      content: ''; position: absolute; top: 0; left: 0; right: 0; 
      height: 4px; background: linear-gradient(90deg, #7c3aed, #db2777); 
      border-radius: 12px 12px 0 0;
  }

  /* Textos */
  .card-title { color: #94a3b8; font-size: 13px; text-transform: uppercase; font-weight: 600; margin-bottom: 8px; }
  .card-value { color: #ffffff; font-size: 32px; font-weight: 800; letter-spacing: -0.5px; }

  /* Ícones de Fundo */
  .icon-bg { position: absolute; right: 20px; top: 24px; opacity: 0.15; }
  svg { width: 48px; height: 48px; stroke: url(#clintGrad); stroke-width: 2; stroke-linecap: round; stroke-linejoin: round; fill: none; }

  /* --- O MÁGICO POP-UP (TOOLTIP) - AGORA PARA BAIXO --- */
  .tooltip-box {
      visibility: hidden;
      opacity: 0;
      position: absolute;
      top: 110%; /* Fica embaixo do cartão */
      left: 50%;
      transform: translateX(-50%);
      background-color: #0f172a;
      color: #cbd5e1;
      text-align: left;
      padding: 12px;
      border-radius: 8px;
      border: 1px solid #7c3aed;
      box-shadow: 0 10px 25px rgba(0,0,0,0.8);
      width: 220px;
      font-size: 11px;
      line-height: 1.4;
      z-index: 1000;
      transition: opacity 0.3s, top 0.3s;
      pointer-events: none;
  }
  
  /* Triângulo apontando para cima */
  .tooltip-box::after {
      content: ''; position: absolute; bottom: 100%; left: 50%;
      margin-left: -6px; border-width: 6px; border-style: solid;
      border-color: transparent transparent #7c3aed transparent;
  }

  /* Mostrar tooltip no hover */
  .clint-card:hover .tooltip-box { visibility: visible; opacity: 1; top: 105%; }
  
  .tooltip-title { color: #db2777; font-weight: bold; font-size: 12px; margin-bottom: 4px; display: block; border-bottom: 1px solid #334155; padding-bottom: 4px;}
</style>
</head>
<body>
  <svg style='width:0;height:0;position:absolute;' aria-hidden='true'><linearGradient id='clintGrad' x1='0%' y1='0%' x2='100%' y2='100%'><stop offset='0%' stop-color='#7c3aed' /><stop offset='100%' stop-color='#db2777' /></linearGradient></svg>

  <div class='clint-cards-wrapper'>
    
    <!-- CARD 1 -->
    <div class='clint-card'>
        <div class='tooltip-box'>
            <span class='tooltip-title'>Regra de Negócio: Leads</span>
            Filtro de integridade aplicado: Exclusão rigorosa de registros com a tag 'DUPLICADO'. Identificamos 15 IDs na base bruta que burlaram o CRM, ajustando a base oficial para 800 leads únicos.
        </div>
        <div class='icon-bg'><svg viewBox='0 0 24 24'><path d='M16 21v-2a4 4 0 0 0-4-4H6a4 4 0 0 0-4 4v2'/><circle cx='9' cy='7' r='4'/><path d='M22 21v-2a4 4 0 0 0-3-3.87'/><path d='M16 3.13a4 4 0 0 1 0 7.75'/></svg></div>
        <div class='card-title'>Total de Leads</div>
        <div class='card-value'>" & vLeads & "</div>
    </div>

    <!-- CARD 2 -->
    <div class='clint-card'>
        <div class='tooltip-box'>
            <span class='tooltip-title'>Regra de Negócio: Vendas</span>
            Contabiliza apenas negócios com status 'won'. Vendas zeradas (R$ 0,00) ou atreladas a contatos duplicados (chaves órfãs) foram removidas para garantir o cálculo exato do Ticket Médio.
        </div>
        <div class='icon-bg'><svg viewBox='0 0 24 24'><polyline points='9 11 12 14 22 4'/><path d='M21 12v7a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11'/></svg></div>
        <div class='card-title'>Vendas</div>
        <div class='card-value'>" & vVendas & "</div>
    </div>

    <!-- CARD 3 -->
    <div class='clint-card'>
        <div class='tooltip-box'>
            <span class='tooltip-title'>Regra de Negócio: Receita</span>
            Receita líquida validada. Reflete o volume financeiro dos 119 negócios reais que possuem rastreabilidade total (Origem e Segmento) após a higienização do modelo de dados (Star Schema).
        </div>
        <div class='icon-bg'><svg viewBox='0 0 24 24'><line x1='12' y1='2' x2='12' y2='22'/><path d='M17 5H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6'/></svg></div>
        <div class='card-title'>Receita Total</div>
        <div class='card-value'>" & vReceita & "</div>
    </div>

    <!-- CARD 4 -->
    <div class='clint-card'>
        <div class='tooltip-box'>
            <span class='tooltip-title'>Regra de Negócio: Ticket</span>
            Cálculo seguro via DAX (Receita Total / Vendas Validadas). Ignora negócios perdidos ('lost') ou de teste, demonstrando o poder de compra real dos clientes convertidos.
        </div>
        <div class='icon-bg'><svg viewBox='0 0 24 24'><polyline points='23 6 13.5 15.5 8.5 10.5 1 18'/><polyline points='17 6 23 6 23 12'/></svg></div>
        <div class='card-title'>Ticket Médio</div>
        <div class='card-value'>" & vTicket & "</div>
    </div>

  </div>
</body>
</html>
"
RETURN htmlString
```

### 4.5. O Mistério dos 815 vs 800 Leads 

Durante a migração da lógica de SQL para o Power BI, a auditoria de modelagem revelou uma falha silenciosa na base de dados bruta fornecida:

A query SQL inicial, que filtrava leads baseando-se apenas na ausência da tag DUPLICADO, retornou 815 leads.

No entanto, ao forçar a integridade de Chave Primária (contact_id únicos) para a construção do Star Schema no Power BI, o número caiu para 800 leads absolutos.

Em vez de mascarar o erro mantendo os 815 (o que causaria um relacionamento de muitos-para-muitos e falhas nos filtros cruzados), optou-se por aplicar a remoção absoluta de duplicatas matemáticas (Table.Distinct) no Power Query.

O Dashboard reflete o número real de 800 leads únicos, garantindo 100% de confiabilidade na taxa de conversão, e gerando um forte insight de negócio: O sistema atual de deduplicação e tagueamento de leads do CRM apresenta falhas e precisa de revisão técnica.


### 4.7. UX Design e Data Storytelling (Tooltips Dinâmicos)

Para garantir que as regras de governança de dados ficassem transparentes para quem consome o relatório, implementei Tooltips HTML/CSS interativos nos cartões de KPI superiores.

Ao invés de criar notas de rodapé longas, o dashboard utiliza o comportamento de Hover (Cursor sobre o cartão). Ao passar o mouse sobre métricas como "Total de Leads" ou "Vendas", um pop-up nativo é acionado via código, explicando a regra de negócio aplicada (ex: a exclusão dos 15 IDs duplicados ou o filtro de valores zerados).

Isso transforma o Dashboard em um ambiente autoexplicativo, reduzindo a sobrecarga cognitiva do usuário e atestando a qualidade técnica da auditoria realizada no backend do modelo.

### 4.8 Construção das Visões e Gráficos

Com a modelagem em Star Schema estabelecida e os KPIs definidos, a interface visual foi desenhada utilizando as boas práticas de Data Storytelling e UX Design (Dark Mode e paleta de cores institucional da Clint). Abaixo, detalho a engenharia por trás de cada visual:

#### 4.8.1 Funil de Vendas Dinâmico (Análise de Conversão)

Para que o funil fosse 100% responsivo aos filtros (Origem e Segmento), evitou-se o uso de uma coluna estática de categorias. Em vez disso, o funil foi arquitetado no Power BI utilizando 7 medidas DAX independentes, plotadas na ordem cronológica exata do processo comercial:

-- 1. Topo do Funil (Base Higienizada)

```dax
Total Leads = COUNTROWS('d_contacts')
```
-- 2. Negócios Gerados

```dax
Funil 02 - Deals = CALCULATE(DISTINCTCOUNT('f_deals'[deal_id]))
```

-- 3. Qualificação de Marketing

```dax
Funil 03 - MQL = CALCULATE(DISTINCTCOUNT('f_pipeline'[deal_id]), 'f_pipeline'[stage] = "MQL")
```

-- 4. Agendamento SDR

```dax
Funil 04 - Reunião Agendada = CALCULATE(DISTINCTCOUNT('f_pipeline'[deal_id]), 'f_pipeline'[stage] = "Reunião Agendada")
```

-- 5. Reuniões Executadas

```dax
Funil 05 - Reunião Realizada = CALCULATE(DISTINCTCOUNT('f_pipeline'[deal_id]), 'f_pipeline'[stage] = "Reunião Realizada")
```

-- 6. Negociação

```dax
Funil 06 - Proposta = CALCULATE(DISTINCTCOUNT('f_pipeline'[deal_id]), 'f_pipeline'[stage] = "Proposta")
```

-- 7. Fundo do Funil (Vendas Ganhas validadas)

```dax
Total Vendas (Won)  = CALCULATE(DISTINCTCOUNT('f_deals'[deal_id]), 'f_deals'[status] = "won", 'f_deals'[valor] > 0)
```
A visualização em ordem do funil expôs uma quebra na linearidade do CRM. Temos 463 Reuniões Realizadas contra apenas 437 MQLs. Isso comprova que a equipe comercial está burlando o funil (registrando atividades retroativamente ou pulando a qualificação obrigatória).




