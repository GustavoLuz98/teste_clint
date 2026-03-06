# Teste Prático - Analista de Dados & Automação | Clint

## 1. Auditoria e Qualidade dos Dados (Etapa 1)
Antes de iniciar as análises de negócio, foi realizada uma auditoria técnica para garantir a integridade do banco de dados analítico.

### 1.1 Inconsistências Identificadas
Durante o processo de exploração e limpeza via SQL, foram detectados os seguintes pontos:

* **Registros Duplicados (Tabela `contacts`):** Identificamos leads duplicados sinalizados pela tag `DUPLICADO` na coluna `tags`. 
    * **Impacto:** Risco de inflar o volume de entrada do funil e distorcer taxas de conversão.
    * **Ação:** Criação de uma camada de visualização (View) que filtra registros com essa tag.

* **Chaves Órfãs (Tabela `pipeline_events`):** Foram encontrados 18 eventos de movimentação no funil vinculados a `deal_id` que não existem na tabela principal de Negócios (`deals`).
    * **Impacto:** Perda de rastreabilidade e métricas de funil sem origem comprovada.
    * **Ação:** Desconsideração desses registros nas métricas de conversão para manter a fidelidade aos negócios validados.

* **Anomalias de Cronologia:** Notou-se que cerca de 50% dos contatos possuem data de criação posterior à criação do negócio.
    * **Decisão Técnica:** Optou-se por manter os dados conforme o original, assumindo que `contact_created_at` pode representar um registro administrativo posterior ou fruto de migração de sistemas legados.

### 1.2 Stack Tecnológica Utilizada
* **Banco de Dados:** SQLite para processamento local.
* **Ferramenta de Gestão:** DB Browser for SQLite.
* **Linguagem:** SQL (DQL e DML) para transformação e limpeza.
