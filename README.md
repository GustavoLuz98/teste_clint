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

## 1.1.3. Verificação de Consistência Temporal
Análise da cronologia entre a criação do contato e a criação do negócio. Foi identificada uma divergência em aproximadamente 50% da base, onde o negócio consta como criado antes do contato.

```sql
SELECT d.deal_id, d.deal_created_at, c.contact_created_at
FROM deals d
JOIN contacts c ON d.contact_id = c.contact_id
WHERE d.deal_created_at < c.contact_created_at;
```

Decisão técnica: Os registros serão mantidos para preservar o volume histórico, considerando uma possível migração de dados ou registro retroativo no CRM.

## 1.1.4. Análise de Valores Inconsistentes
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
* **Ferramenta de Gestão:** DB Browser for SQLite.
* **Linguagem:** SQL (DQL e DML) para transformação e limpeza.
