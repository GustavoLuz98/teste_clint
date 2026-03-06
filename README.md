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

## 3.1 Receita total = R$ 2.143.280,00 e o Ticket Médio = R$ 17.713,06
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

## 3.2 Receita por Origem de Marketing

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

## 3.3 Receita por Segmento

| Segmento | Receita Total | Ticket Médio |
|----------|---------------|--------------|
|Educação|	R$ 410.723|	19558.2380952381|
|Advocacia|	R$ 360.518|	18025.9|
|Saúde|	R$ 299.775|	21412.5|
|Serviços|	R$ 288.564|	19237.6|
|Agência|	R$ 271.857|	19418.3571428571|
|Infoprodutor|	R$ 194.011|	12934.0666666667|
|Indústria|	R$ 166.907|	16690.7|
|E-commerce|	R$ 117.524|	11752.4|

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

## 3.4 Tempo Médio de Ciclo de Vendas = 67 Dias

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

## 3.5 Performance dos Closers (Receita Por Vendedor)

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

