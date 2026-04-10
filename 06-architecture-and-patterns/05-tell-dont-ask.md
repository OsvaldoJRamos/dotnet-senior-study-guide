# Tell, Don't Ask

## Summary

The "Tell, Don't Ask" principle helps us structure code so that objects themselves are responsible for their operations, instead of exposing data for another class to make decisions. This improves cohesion, reduces coupling, and makes the code more aligned with Object-Oriented Programming principles.

## Example

We have a BankAccount class with a BALANCE property. Instead of having a BankAccountService that retrieves the balance value (BankAccount.Balance) and validates whether the withdrawal can be made, this logic should be inside the entity.

### Wrong (Ask):
```csharp
public class BankAccountService
{
    public void ProcessWithdrawal(BankAccount account, decimal amount)
    {
        if (account.Balance >= amount)
        {
            account.Withdraw(amount);
        }
        else
        {
            Console.WriteLine("Insufficient balance!");
        }
    }
}
```

### Correct (Tell):
```csharp
public class BankAccount
{
    private decimal _balance;
    private decimal _dailyLimit;
    private decimal _withdrawnToday;

    public BankAccount(decimal initialBalance, decimal dailyLimit)
    {
        _balance = initialBalance;
        _dailyLimit = dailyLimit;
        _withdrawnToday = 0;
    }

    public void Withdraw(decimal amount)
    {
        if (_withdrawnToday + amount > _dailyLimit)
            throw new InvalidOperationException("Daily withdrawal limit exceeded!");
        if (_balance < amount)
            throw new InvalidOperationException("Insufficient balance!");
        _balance -= amount;
        _withdrawnToday += amount;
    }

    public decimal GetBalance() => _balance;
}
```

The business logic (balance and limit validation) stays **inside the entity**, not in an external service.

---

[← Previous: Object Calisthenics](04-object-calisthenics.md) | [Back to index](README.md) | [Next: SAGA Pattern →](06-saga-pattern.md)
