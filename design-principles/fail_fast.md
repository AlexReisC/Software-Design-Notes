# Fail Fast
**Definição:** Sistema detecta e reporta erro o mais cedo possível. Falha ruidosa imediata > falha silenciosa tardia. Quanto mais tarde o erro aparece, mais difícil achar a causa.

## Exemplos de código

❌ RUIM - falha silenciosa e tardia

```java
public class OrderService {

    public void processOrder(Order order) {
        // não valida entrada, segue em frente
        double total = calculateTotal(order);
        Payment payment = paymentGateway.charge(total);
        // payment pode ser null se gateway falhou silenciosamente
        
        String confirmationId = payment.getConfirmationId(); // NullPointerException AQUI
        // erro aconteceu no gateway, explode 3 passos depois
        // stack trace aponta linha errada
        // causa real: perdida
        
        emailService.send(order.getCustomer().getEmail(), confirmationId);
        repository.save(order);
    }
}
```

Problema:
- `order` nulo → `NullPointerException` em `calculateTotal`
- Gateway falha → retorna null → NPE 3 passos depois
- Stack trace aponta sintoma, não causa
- Debug: rastreia pra trás, sem contexto

✅ BOM - falha imediata com contexto

```java
public class OrderService {

    public void processOrder(Order order) {
        // falha no boundary, antes de qualquer processamento
        validateOrder(order);

        double total = calculateTotal(order);

        Payment payment = paymentGateway.charge(total);
        validatePayment(payment, total); // falha imediata se gateway retornou lixo

        String confirmationId = payment.getConfirmationId();
        emailService.send(order.getCustomer().getEmail(), confirmationId);
        repository.save(order);
    }

    private void validateOrder(Order order) {
        Objects.requireNonNull(order, "Order must not be null");
        Objects.requireNonNull(order.getCustomer(), "Order must have a customer");
        Objects.requireNonNull(order.getItems(), "Order must have items");

        if (order.getItems().isEmpty())
            throw new InvalidOrderException("Order must have at least one item");

        if (order.getCustomer().getEmail() == null)
            throw new InvalidOrderException("Customer must have an email");
    }

    private void validatePayment(Payment payment, double expectedAmount) {
        Objects.requireNonNull(payment, "Payment gateway returned null");

        if (payment.getStatus() != PaymentStatus.APPROVED)
            throw new PaymentFailedException(
                "Payment not approved. Status: " + payment.getStatus()
            );

        if (payment.getConfirmationId() == null)
            throw new PaymentFailedException("Payment approved but no confirmation ID");
    }
}
```

---

Fail Fast em construção de objeto

```java
// ❌ objeto inválido circula pelo sistema
public class User {
    private String email;
    private String name;

    public User(String name, String email) {
        this.name = name;
        this.email = email; // aceita null, vazio, lixo
    }
}

// explode lá na frente, sem contexto
emailService.send(user.getEmail(), message); // NPE aqui
```

```java
// ✅ construtor falha rápido - objeto válido ou não existe
public class User {
    private final String name;
    private final String email;

    public User(String name, String email) {
        this.name = requireNonBlank(name, "name");
        this.email = requireValidEmail(email);
    }

    private String requireNonBlank(String value, String field) {
        if (value == null || value.isBlank())
            throw new IllegalArgumentException(field + " must not be blank");
        return value;
    }

    private String requireValidEmail(String email) {
        if (email == null || !email.contains("@"))
            throw new IllegalArgumentException("Invalid email: " + email);
        return email;
    }
}
// Se User existe → é válido. Garantia de invariante.
```

---

Fail Fast com assertions em domníno crítico

```java
// Invariante de negócio, nunca deve ser violada
public class BankAccount {
    private double balance;

    public void withdraw(double amount) {
        if (amount <= 0)
            throw new IllegalArgumentException("Withdrawal amount must be positive: " + amount);

        if (amount > balance)
            throw new InsufficientFundsException(
                "Cannot withdraw %.2f. Balance: %.2f".formatted(amount, balance)
            );

        balance -= amount;

        // assert pós-condição, detecta bug de lógica imediatamente
        assert balance >= 0 : "Balance went negative after withdrawal - logic bug";
    }
}
```

---

Fail Fast na inicialização - Spring

```java
// ❌ config inválida → explode na primeira requisição em prod
@Service
public class PaymentService {
    @Value("${payment.api.key}")
    private String apiKey; // null se não configurado - falha em runtime

    public void charge(double amount) {
        client.post(apiKey, amount); // NPE aqui, sob carga
    }
}
```
```java
// ✅ falha no startup, antes de aceitar requisição
@Service
public class PaymentService {
    private final String apiKey;

    public PaymentService(@Value("${payment.api.key}") String apiKey) {
        if (apiKey == null || apiKey.isBlank())
            throw new IllegalStateException(
                "payment.api.key not configured. Set it in application.properties"
            );
        this.apiKey = apiKey;
    }
}
// App não sobe → dev sabe imediatamente → não chega em prod quebrado
```

## FAIL FAST vs FAST SAFE

```
Fail Fast  → detecta erro cedo, para, reporta
             ideal: lógica de negócio, dados críticos, config

Fail Safe  → detecta erro, continua com fallback
             ideal: features opcionais, degradação graciosa

// Fail Safe legítimo
public String getUserAvatar(UUID userId) {
    try {
        return avatarService.getUrl(userId);
    } catch (AvatarServiceException e) {
        log.warn("Avatar service unavailable for user {}. Using default.", userId);
        return DEFAULT_AVATAR_URL; // falha graciosa - não é dado crítico
    }
}
```

Não confunde: Fail Safe em dado crítico = silencia erro real.

## REGRAS CORE
| Regra | Detalhe |
| ----- | ------- |
| Valida no boundary | Entrada da API, construtor, início do método público |
| Erro com contexto | `"Invalid email: " + email` > `"Invalid input"` |
| Nunca retorna null | `Optional<T>` ou lança exceção |
| Config valida no startup | App não sobe com config inválida |
| Pós-condição crítica | Assert após operação financeira/crítica |
| Log antes de relançar | Contexto não se perde na stack |
