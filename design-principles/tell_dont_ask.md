# Tell, don't ask

**Definição:** Diz ao objeto o que fazer. Não pergunta estado pra decidir por ele. Decisão vive no objeto, não no caller.

## Problema Raiz
"Ask" = caller extrai dado → decide fora → age. Lógica que devia estar no objeto vaza pro caller. Encapsulamento quebrado.

## Exemplos de código

❌ RUIM - pergunta estado, decide fora

```java
// Caller PERGUNTA estado e DECIDE
public class OrderService {

    public void processOrder(Order order) {

        // pergunta status
        if (order.getStatus() == OrderStatus.PENDING) {

            // pergunta itens
            if (order.getItems().isEmpty()) {
                throw new InvalidOrderException("Empty order");
            }

            // pergunta total
            double total = 0;
            for (Item item : order.getItems()) {
                // pergunta preço e quantidade
                total += item.getPrice() * item.getQuantity();
            }
            order.setTotal(total);

            // pergunta se premium
            if (order.getCustomer().isPremium()) {
                order.setTotal(order.getTotal() * 0.9);
            }

            order.setStatus(OrderStatus.CONFIRMED);
        }
    }
}
```

Problema:
- Lógica de Order vive em OrderService
- Regra de desconto duplicável em qualquer caller
- Order é objeto passivo, dados sem comportamento
- Encapsulamento violado

✅ LIMPO - diz o que fazer

```java
// Order SABE fazer - caller só diz
public class Order {
    private OrderStatus status;
    private double total;
    private List<Item> items;
    private Customer customer;

    // caller diz "confirma" - Order decide como
    public void confirm() {
        if (status != OrderStatus.PENDING)
            throw new InvalidOrderException("Only pending orders can be confirmed");
        if (items.isEmpty())
            throw new InvalidOrderException("Cannot confirm empty order");

        this.total = calculateTotal();
        applyDiscountIfEligible();
        this.status = OrderStatus.CONFIRMED;
    }

    // lógica vive aqui, não no caller
    private double calculateTotal() {
        return items.stream()
            .mapToDouble(item -> item.getPrice() * item.getQuantity())
            .sum();
    }

    private void applyDiscountIfEligible() {
        if (customer.isPremium())
            this.total *= 0.9;
    }
}

// Caller - diz, não pergunta
public class OrderService {
    public void processOrder(Order order) {
        order.confirm(); // uma linha. Order sabe o resto.
        repository.save(order);
        notificationService.sendConfirmation(order);
    }
}
```

---

❌ ASK - padrão anêmico

```java
// Modelo anêmico - dado sem comportamento
public class User {
    private boolean active;
    private LocalDate lastLogin;
    private int loginAttempts;

    public boolean isActive() { return active; }
    public LocalDate getLastLogin() { return lastLogin; }
    public int getLoginAttempts() { return loginAttempts; }
    public void setActive(boolean active) { this.active = active; }
    public void setLoginAttempts(int attempts) { this.loginAttempts = attempts; }
}

// Lógica espalhada em qualquer service que "pergunta"
public class AuthService {
    public void lockIfNeeded(User user) {
        if (user.getLoginAttempts() >= 5 && user.isActive()) {
            user.setActive(false); // setter livre - sem regra
        }
    }
}

public class AdminService {
    public void checkAndLock(User user) {
        if (user.getLoginAttempts() > 5) { // regra duplicada, threshold diferente!
            user.setActive(false);
        }
    }
}
```

✅ TELL - modelo rico

```java
// User SABE suas regras
public class User {
    private boolean active;
    private LocalDate lastLogin;
    private int loginAttempts;

    private static final int MAX_ATTEMPTS = 5;

    public void recordFailedLogin() {
        this.loginAttempts++;
        if (loginAttempts >= MAX_ATTEMPTS)
            lock(); // User decide quando travar
    }

    public void recordSuccessfulLogin() {
        this.loginAttempts = 0;
        this.lastLogin = LocalDate.now();
    }

    private void lock() {
        this.active = false;
    }

    public void unlock() {
        this.active = true;
        this.loginAttempts = 0;
    }

    public boolean isLocked() { return !active; }
}

// Caller: diz o que aconteceu. User decide consequência.
public class AuthService {
    public AuthResult authenticate(String email, String password) {
        User user = repository.findByEmail(email).orElseThrow();

        if (!passwordEncoder.matches(password, user.getPasswordHash())) {
            user.recordFailedLogin(); // diz - não pergunta
            repository.save(user);
            return AuthResult.FAILED;
        }

        user.recordSuccessfulLogin(); // diz - não pergunta
        repository.save(user);
        return AuthResult.SUCCESS;
    }
}
```

---

TELL DON'T ASK + LEI DE DEMETER

```java
// ❌ Ask + LoD violation: pergunta E navega estrutura
if (order.getCustomer().getAddress().getCountry().isEuMember()) {
    applyVat(order);
}

// ✅ Tell - Order sabe responder sobre si mesmo
if (order.requiresVat()) { // Order pergunta ao Customer internamente
    order.applyVat();
}

// Order encapsula navegação
public class Order {
    public boolean requiresVat() {
        return customer.isInEuCountry(); // delega, não expõe estrutura
    }

    public void applyVat() {
        this.total *= 1.23;
    }
}
```

## Exceção legítima

```java
// Às vezes "ask" é ok - quando caller SÓ precisa do dado
// sem tomar decisão sobre o objeto

// ✅ ok - caller usa dado pra próprio contexto, não pra decidir pelo objeto
String email = user.getEmail();
emailSender.send(email, message); // não decide sobre User

// ✅ ok - query pura, sem side effect
double balance = account.getBalance();
report.addRow(account.getId(), balance); // não age sobre Account
```

Regra: Ask é OK quando caller usa dado **pro próprio trabalho**. Ruim quando usa dado **pra decidir pelo objeto**.

