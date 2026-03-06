# Manipulações de dados em SQL

## 1. Tratamento de Leads Duplicados
Identificação e exclusão lógica de registros marcados com a tag "DUPLICADO" na tabela de contatos.

-- Consulta para validar volume de duplicatas antes da análise
SELECT COUNT(*) AS total_duplicados
FROM contacts 
WHERE tags LIKE '%DUPLICADO%';

-- Criação de uma View para garantir que as próximas etapas usem apenas dados únicos
CREATE VIEW vw_contacts_unicos AS
SELECT * FROM contacts 
WHERE tags NOT LIKE '%DUPLICADO%' OR tags IS NULL;
