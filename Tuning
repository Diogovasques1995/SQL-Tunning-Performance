# 🚀 SQL Performance & Query Tuning

### Otimização de Consultas no Projeto Portal da Transparência / ANAC

---

## 🎯 Objetivos do Aprendizado

Esta seção documenta o processo de análise de performance e tunagem de consultas T-SQL.
O foco foi identificar gargalos de processamento, reduzir o custo de execução (Logical Reads)
e otimizar o tempo de resposta do banco de dados.

---

## 🔧 Ferramentas Utilizadas

- **Execution Plan:** Análise visual do plano de execução para identificar Clustered Index Scans e Key Lookups.
- **Statistics IO/TIME:** Medição do custo de leitura em disco e tempo de CPU.
- **Database Engine Tuning Advisor:** Para sugestões de novos índices.

---

## 📊 Análises e Comentários Técnicos

### 1. Otimização de Filtros (SARGability)

❌ **Antes: Lento**
```sql
-- Consulta Original com função no WHERE (impede uso de índice)
SELECT nome_viajante, valor_total_diarias 
FROM Viagens 
WHERE YEAR(data_inicio) = 2026;
```

> 💡 **Comentário de Tuning:** O uso da função `YEAR()` na coluna de filtro causa um *Index Scan*, forçando o SQL a ler toda a tabela. Reescrita para usar um intervalo de datas, permitindo o *Index Seek*.

✅ **Depois: Rápido**
```sql
-- Consulta Otimizada (SARGable)
SELECT nome_viajante, valor_total_diarias 
FROM Viagens 
WHERE data_inicio >= '2026-01-01' AND data_inicio <= '2026-12-31';
```

---

### 2. Redução de Key Lookups

```sql
-- Query identificada com alto custo de I/O
SELECT nome_viajante, motivo, orgao_solicitante 
FROM Viagens WITH(INDEX(IX_DataInicio))
WHERE data_inicio = '2026-05-15';
```

> 💡 **Comentário de Tuning:** O plano de execução revelou um *Key Lookup*. A solução foi aplicar um índice não-clusterizado incluindo as colunas de retorno (Covering Index), eliminando a necessidade de buscar dados na tabela principal.

---

### 3. Otimização de Joins com CTE

```sql
WITH ViagensFiltradas AS (
    SELECT id_processo_viagem, nome_viajante 
    FROM Viagens 
    WHERE orgao_solicitante = 'Ministério da Saúde'
)
SELECT VF.nome_viajante, T.destino_cidade
FROM ViagensFiltradas VF
JOIN Trechos T ON VF.id_processo_viagem = T.id_processo_viagem;
```

> 💡 **Comentário de Tuning:** Ao filtrar os dados dentro de um CTE antes de realizar o JOIN com a tabela de Trechos, reduzimos o volume de dados processados na junção, melhorando significativamente a velocidade da consulta em grandes volumes.

---

## ✅ Resultados Obtidos

| Métrica | Original | Otimizada |
|---|---|---|
| Leituras Lógicas | ~15.400 | ~420 |
| Tempo de Execução | 2.4s | 0.1s |
