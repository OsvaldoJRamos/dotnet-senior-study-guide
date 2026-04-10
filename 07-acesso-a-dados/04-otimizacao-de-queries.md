# Otimizacao de Queries SQL

## O que faz uma query SQL ser lenta?

Combinacao de fatores de modelagem, indexacao, volume de dados e forma como a query e escrita:

1. **Falta ou uso incorreto de indices** - colunas em WHERE, JOIN, ORDER BY, GROUP BY sem indice adequado
2. **Funcoes nas colunas** - impede uso de indices (consulta nao-sargavel)
3. **Joins mal planejados** - muitas tabelas, cardinalidade errada, joins em colunas nao indexadas
4. **Volume de dados alto** sem paginacao ou particionamento
5. **SELECT \* desnecessario** - traz mais dados do que o necessario
6. **Subqueries e CTEs mal usadas** - quando poderiam ser joins mais eficientes
7. **Estatisticas desatualizadas** - otimizador escolhe planos ruins
8. **Bloqueios e concorrencia** - locks, transacoes longas, isolamento mal configurado
9. **Infraestrutura** - I/O lento, pouca memoria para cache

### Abordagem pratica

Em producao:
1. Olhar o **plano de execucao**
2. Verificar metricas de **tempo e I/O**
3. Validar se a query esta **usando os indices esperados**
4. Ajustar query, indices ou modelo, **medindo impacto antes e depois**

## Parameter Sniffing (SQL Server)

### O que e

O SQL Server usa Parameter Sniffing para otimizar queries parametrizadas:

1. Na **primeira execucao**, pega os valores reais dos parametros
2. Gera um **plano de execucao otimizado** para esses valores
3. **Cacheia** o plano para reutilizar nas proximas execucoes

### O problema

Se os proximos parametros forem **muito diferentes** dos primeiros, o plano cacheado pode ser **pessimo** para os novos valores.

Sintoma classico: query funciona rapido no SSMS (SQL local) e **trava** rodando pela aplicacao.

### Solucao: OPTION (RECOMPILE)

```sql
SELECT * FROM Pedidos
WHERE ClienteId = @ClienteId
AND DataCriacao BETWEEN @DataInicio AND @DataFim
OPTION (RECOMPILE)  -- forca novo plano a cada execucao
```

### Trade-off

- **Custo**: ~10-50ms de overhead de compilacao por query
- **Beneficio**: evita queries que travavam por 30-60 segundos
- **Resultado tipico**: query de 45s cai para ~300ms

### Quando NAO usar RECOMPILE em tudo

- Queries simples e rapidas (<10ms) - o overhead de compilacao pode dobrar o tempo
- Consome CPU extra do SQL Server
- So e necessario quando ha problemas reais de parameter sniffing

## Consultas nao-sargaveis

Uma consulta e **sargavel** (Search ARGument ABLE) quando o otimizador consegue usar indices para filtrar.

```sql
-- NAO SARGAVEL (funcao na coluna impede uso de indice)
WHERE ISNUMERIC(Quantity) = 1
WHERE YEAR(DataCriacao) = 2024
WHERE UPPER(Nome) = 'JOAO'

-- SARGAVEL (indice pode ser usado)
WHERE Quantity IS NOT NULL AND Quantity > 0
WHERE DataCriacao >= '2024-01-01' AND DataCriacao < '2025-01-01'
WHERE Nome = 'JOAO'  -- com collation case-insensitive
```

### Exemplo real

```sql
-- Problematico: ISNUMERIC() impede uso de indice
SELECT SUM(CAST(RemainingQuantity AS decimal))
FROM OrderProductDetails opd WITH (NOLOCK)
WHERE opd.OrderId = os1.Id
  AND opd.Quantity != ''
  AND ISNUMERIC(opd.Quantity) = 1  -- nao-sargavel!

-- Plano bom: processa apenas as rows necessarias
-- Plano ruim: calcula ISNUMERIC() para TODAS as rows
```

**Solucao**: usar coluna computada com indice ou corrigir o tipo de dado no banco.

## Dicas gerais de otimizacao

1. **Crie indices** nas colunas usadas em WHERE, JOIN, ORDER BY - mas nao exagere (indices demais prejudicam escrita)
2. **Evite SELECT \*** - selecione apenas as colunas necessarias
3. **Use paginacao** (OFFSET/FETCH ou keyset pagination)
4. **Atualize estatisticas** do banco regularmente
5. **Use NOLOCK com cautela** - pode causar dirty reads
6. **Prefira EXISTS a IN** para subqueries com muitos resultados
7. **Evite cursors** - use operacoes baseadas em conjunto (SET-based)

---

[← Anterior: Bancos de Dados](03-bancos-de-dados.md) | [Voltar ao índice](README.md)
