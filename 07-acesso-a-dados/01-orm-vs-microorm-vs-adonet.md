# ORM, Micro ORM ou ADO.NET Puro

## Resumo: quando usar cada um

| Critério | ORM (EF Core) | Micro ORM (Dapper) | ADO.NET Puro |
|---|---|---|---|
| Produtividade | Alta | Média | Baixa |
| Performance | Média | Alta | Máxima |
| Controle do SQL | Baixo | Alto | Total |
| Migrations | Sim | Não | Não |
| Facilidade de testes | Boa | Boa | Difícil |
| Manutenção a longo prazo | Alta | Média | Baixa |
| Curva de aprendizado | Média | Baixa | Alta |

### Conclusão prática:
- **EF Core (ORM):** ideal para 80% dos sistemas comerciais, CRUDs, APIs REST, aplicações corporativas
- **Dapper (Micro ORM):** ideal para sistemas de performance crítica, relatórios pesados, microservices, controle total
- **ADO.NET Puro:** último recurso; use somente quando você realmente precisar de controle de baixo nível

---

## ORM (Entity Framework Core)

### O que é?
Ferramenta que mapeia objetos C# para tabelas do banco de dados, escondendo os detalhes do SQL. Permite trabalhar com banco como se fosse com objetos.

### Quando usar?
- Projetos médios a grandes com muitas entidades
- Você precisa de produtividade e pouca repetição de código SQL
- Regras de negócio complexas, mas sem extrema necessidade de performance

### Vantagens:
- Alta produtividade (CRUD quase automático)
- Migrations integradas
- Lazy/eager loading, tracking, LINQ
- Boa integração com ASP.NET Core e DI

### Desvantagens:
- Overhead de performance
- Pouco controle sobre queries geradas
- Complexidade em otimizações avançadas

### Usos reais:
- Sistemas administrativos
- APIs RESTful
- Aplicações com modelo rico e regras de negócio

---

## Micro ORM (Dapper, RepoDb)

### O que é?
Uma ferramenta leve que mapeia objetos rapidamente, mas exige que você escreva as queries SQL.

### Quando usar?
- Projetos onde performance é crítica
- Você prefere escrever SQL manualmente
- Precisa de mais controle e agilidade
- Sistemas com grande volume de dados ou integrações com bancos legados

### Vantagens:
- Super rápido
- Simples de usar
- Muito controlado
- Pode ser usado junto com ADO.NET puro

### Desvantagens:
- Sem migrations
- Sem tracking de objetos
- Menos produtividade em projetos grandes

### Usos reais:
- APIs de alta performance
- Relatórios complexos
- Serviços que fazem acesso massivo ao banco
- Microservices com banco específico

---

## ADO.NET Puro

### O que é?
É a base de tudo: classes como `SqlConnection`, `SqlCommand`, `SqlDataReader`, etc. Permite controle total sobre o acesso ao banco.

### Quando usar?
- Você precisa de máxima performance
- Queries extremamente complexas, tuning manual
- Integração com bancos não relacionais
- Projetos legados que já usam ADO.NET
- Cenários com baixo nível de abstração (ex: jogos, dispositivos embarcados)

### Vantagens:
- Máximo controle
- Zero abstração
- Nenhuma mágica por trás

### Desvantagens:
- Muito verboso
- Baixa produtividade
- Fácil cometer erros
- Requer manutenção manual de SQL e mapeamento

### Exemplo:
```csharp
var clientes = new List<Cliente>();

using var conn = new SqlConnection(connectionString);
conn.Open();

var cmd = new SqlCommand("SELECT * FROM Clientes WHERE Ativo = 1", conn);
using var reader = cmd.ExecuteReader();

while (reader.Read())
{
    clientes.Add(new Cliente
    {
        Id = (int)reader["Id"],
        Nome = reader["Nome"].ToString()
    });
}
```

### Usos reais:
- Sistemas embarcados
- Ferramentas internas com foco em performance
- Aplicações com tuning avançado (caching, indexes, locks)

---

[Voltar ao índice](README.md) | [Próximo: Entity Framework →](02-entity-framework.md)
