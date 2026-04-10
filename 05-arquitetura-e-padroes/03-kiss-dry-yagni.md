# KISS, DRY e YAGNI

## YAGNI (You Aren't Gonna Need It)

Recursos só devem ser desenvolvidos quando estritamente necessários, evitando assim desperdício de tempo.

Evita a elaboração de implementações complexas, pensando que no futuro pode haver outras necessidades. E essas necessidades podem nunca chegar. Deixe para desenvolver soluções complexas quando houver a necessidade. Se não houver, opte pelo simples.

**Exemplo:** foi solicitada a implementação de uma forma de pagamento por cartão de crédito. O desenvolvedor resolveu implementar uma solução que pode no futuro tratar diversos outros tipos (como boleto e pix), porém passa-se os anos e o sistema nunca precisou disso.

## KISS (Keep It Simple, Stupid)

Faça tudo da forma mais simples possível, para que todos consigam entender rapidamente. Evite ao máximo complexidade desnecessária.

```csharp
// KISS violado - complexidade desnecessária
public bool IsAdult(int age)
{
    if (age >= 18)
    {
        return true;
    }
    else
    {
        return false;
    }
}

// KISS aplicado
public bool IsAdult(int age) => age >= 18;
```

## DRY (Don't Repeat Yourself)

Evite sempre código duplicado. Escreva códigos reutilizáveis, modulares e abstratos o suficiente para que possam ser usados em várias partes do código.

```csharp
// DRY violado - validação duplicada
public void CriarUsuario(string email)
{
    if (!email.Contains("@")) throw new Exception("Email inválido");
    // ...
}

public void AtualizarEmail(string email)
{
    if (!email.Contains("@")) throw new Exception("Email inválido");
    // ...
}

// DRY aplicado - validação centralizada
public static class EmailValidator
{
    public static void Validar(string email)
    {
        if (!email.Contains("@")) throw new Exception("Email inválido");
    }
}
```

**Cuidado:** DRY não significa que qualquer código parecido deve ser abstraído. Se duas coisas são parecidas **por coincidência** mas mudam por motivos diferentes, mantê-las separadas pode ser o correto.

---

[← Anterior: Design Patterns](02-design-patterns.md) | [Voltar ao índice](README.md) | [Próximo: Object Calisthenics →](04-object-calisthenics.md)
