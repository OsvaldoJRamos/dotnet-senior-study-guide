# KISS, DRY and YAGNI

## YAGNI (You Aren't Gonna Need It)

Features should only be developed when strictly necessary, thus avoiding wasted time.

Avoid building complex implementations thinking that in the future there may be other needs. Those needs may never come. Save complex solutions for when the need actually arises. If there is no need, go with the simple approach.

**Example:** a credit card payment method was requested. The developer decided to implement a solution that could handle various other payment types in the future (such as bank slip and pix), but years went by and the system never needed any of that.

## KISS (Keep It Simple, Stupid)

Do everything in the simplest way possible, so that everyone can understand it quickly. Avoid unnecessary complexity as much as possible.

```csharp
// KISS violated - unnecessary complexity
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

// KISS applied
public bool IsAdult(int age) => age >= 18;
```

## DRY (Don't Repeat Yourself)

Always avoid duplicated code. Write reusable, modular, and abstract enough code so it can be used in multiple parts of the codebase.

```csharp
// DRY violated - duplicated validation
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

// DRY applied - centralized validation
public static class EmailValidator
{
    public static void Validar(string email)
    {
        if (!email.Contains("@")) throw new Exception("Email inválido");
    }
}
```

**Caution:** DRY does not mean that any similar-looking code should be abstracted. If two things look similar **by coincidence** but change for different reasons, keeping them separate may be the right choice.

---

[← Previous: Design Patterns](02-design-patterns.md) | [Back to index](README.md) | [Next: Object Calisthenics →](04-object-calisthenics.md)
