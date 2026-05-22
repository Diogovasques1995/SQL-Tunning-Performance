# Performance Tuning no PostgreSQL (AWS RDS)

Este repositório documenta um laboratório prático de otimização de consultas e análise de planos de execução (*Query Tuning*) num ambiente de produção na nuvem, utilizando uma instância do **PostgreSQL** alojada no **AWS RDS (Relational Database Service)** e o **pgAdmin 4**.

---

## 🎯 O Cenário & O Desafio

O objetivo foi otimizar uma consulta analítica que filtra dados de um grande volume de registos na tabela `projeto_aws.pedidos` com base num intervalo de datas específico e num estado de envio (`status = 'Enviado'`).

Em sistemas de grande escala (como monitorização de telemetria ou e-commerce), consultas sem indexação forçam o motor do banco de dados a realizar uma varredura sequencial completa (*Sequential Scan*), gerando latência e consumo desnecessário de I/O e CPU na nuvem.

---

## 🔬 Investigação e Diagnóstico (Métrica Inicial)

Utilizando o comando descritivo mais poderoso do PostgreSQL — o `EXPLAIN ANALYZE` —, conseguimos executar a consulta capturando o comportamento real do motor de busca e o tempo exato de execução antes de qualquer alteração na estrutura.

```sql
EXPLAIN ANALYZE
SELECT * FROM projeto_aws.pedidos
WHERE data_pedido BETWEEN '2026-04-01 00:00:00' AND '2026-04-15 23:59:59'
  AND status = 'Enviado';
```

---

## 🛠️ Estratégia de Tuning: Evolução da Indexação

### 1ª Abordagem: Índice Simples de Coluna (Single-Column Index)

Criámos inicialmente um índice focado apenas na coluna de data para mitigar a leitura sequencial:

```sql
CREATE INDEX idx_pedidos_data ON projeto_aws.pedidos (data_pedido);
```

**Resultado do Plano (`EXPLAIN ANALYZE`):**

- O banco abandonou o `Seq Scan` e passou a utilizar um `Bitmap Index Scan`.
- O problema: o índice resolveu a busca por data, mas o Postgres ainda teve de descartar **12.443 linhas** na tabela principal (`Rows Removed by Filter`) porque o índice não cobria a coluna `status`. O tempo de execução fixou-se em **5.886 ms**.

---

### 2ª Abordagem: O Índice Composto Cirúrgico (Composite Index)

Para atingir a eficiência máxima, eliminámos o índice anterior e desenhámos um **Índice Composto (Multi-column)** que resolve o filtro completo diretamente na estrutura de memória, cobrindo a data e o estado em simultâneo.

```sql
-- Removendo o índice simples anterior
DROP INDEX projeto_aws.idx_pedidos_data;

-- Criando o índice composto de alta performance
CREATE INDEX idx_pedidos_data_status
ON projeto_aws.pedidos (data_pedido, status);
```

---

## 📊 Resultados Finais após o Índice Composto

Ao reexecutar o `EXPLAIN ANALYZE`, o plano de execução validou a otimização com precisão cirúrgica:

```
Bitmap Heap Scan on pedidos (actual time=3.886..5.335 rows=4109.00 loops=1)
  Index Cond: ((data_pedido >= '2026-04-01 00:00:00')
           AND (data_pedido <= '2026-04-15 23:59:59')
           AND (status = 'Enviado'))
  -> Bitmap Index Scan on idx_pedidos_data_status (actual time=3.791..3.792 rows=4109.00 loops=1)
Planning Time: 0.212 ms
Execution Time: 6.208 ms
```

---

## 📈 Impacto Técnico e de Negócio

1. **Desperdício Zero de Processamento:** A linha `Rows Removed by Filter` sumiu por completo. O motor da AWS foi ao índice composto e extraiu cirurgicamente apenas as **4.109 linhas exatas** que atendiam a todos os critérios.

2. **Alta Performance com Leitura Física:** Mesmo com o banco a efetuar uma leitura física direta no disco (registado pelo log `read=80` e `I/O Timings`), a consulta respondeu em impressionantes **6.208 milissegundos**.

3. **FinOps e Escalabilidade:** Demonstrámos na prática como reduzir o custo computacional de um servidor AWS RDS otimizando a lógica física do banco, poupando recursos financeiros e de hardware.

---

## 🧰 Tecnologias Utilizadas

| Tecnologia | Papel |
|---|---|
| **PostgreSQL** | Motor de banco de dados relacional |
| **AWS RDS** | Infraestrutura de banco de dados na nuvem |
| **pgAdmin 4** | Interface gráfica de administração |
| **Query Tuning** | Análise e otimização de consultas SQL |
| **Indexing** | Estratégias de indexação (simples e composta) |
| **Query Plan Analysis** | Interpretação de planos de execução via `EXPLAIN ANALYZE` |
