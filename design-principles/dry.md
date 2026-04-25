# DRY - DON'T REPEAT YOURSELF

**Definição:** Todo conhecimento tem **representação única** no sistema. Não é "não copiar código", é não duplicar **conhecimento/decisão**.'

> "Every piece of knowledge must have a single, unambiguous, authoritative representation within a system." — Andy Hunt & Dave Thomas, The Pragmatic Programmer

## Conceito

Duplicação cria:
- ambiente inconsistente
- múltiplos pontos de mudança

Nuance importante:
Dois códigos semelhantes != duplicação, a menos que representem a mesma regra.

## Exemplos de código

DUPLICAÇÃO DE CÓDIGO - caso óbvio

```java
// ❌ mesma lógica em dois lugares
public class OrderService {
    public double calculateTotal(List<Item> items) {
        double total = 0;
        for (Item item : items)
            total += item.getPrice() * item.getQuantity();
        return total;
    }
}

public class InvoiceService {
    public double calculateTotal(List<Item> items) {
        double total = 0;
        for (Item item : items)
            total += item.getPrice() * item.getQuantity(); // cópia exata
        return total;
    }
}
```

Regra muda → desconto por quantidade → atualiza dois lugares. Esquece um → bug silencioso.

```java
// ✅ conhecimento centralizado
public class PriceCalculator {
    public double calculateTotal(List<Item> items) {
        return items.stream()
            .mapToDouble(item -> item.getPrice() * item.getQuantity())
            .sum();
    }
}

// ambos delegam
public class OrderService {
    private final PriceCalculator calculator;
    public double calculateTotal(List<Item> items) {
        return calculator.calculateTotal(items);
    }
}
```

---

DUPLICAÇÃO DE CONHECIMENTO - caso sutil

```java
// ❌ regra de negócio espalhada
public class UserService {
    public boolean isPremium(User user) {
        return user.getMonthlySpend() > 1000.0; // regra aqui
    }
}

public class DiscountService {
    public double getDiscount(User user) {
        if (user.getMonthlySpend() > 1000.0) // mesma regra aqui
            return 0.15;
        return 0.0;
    }
}

public class ShippingService {
    public boolean freeShipping(User user) {
        return user.getMonthlySpend() > 1000.0; // e aqui
    }
}
```

Threshold muda pra `1500.0` → caça em 3 classes. Esquece `ShippingService` → bug em produção.

```java
// ✅ decisão única
public class PremiumPolicy {
    private static final double PREMIUM_THRESHOLD = 1000.0;

    public boolean isPremium(User user) {
        return user.getMonthlySpend() > PREMIUM_THRESHOLD;
    }
}

// todos delegam pra mesma decisão
public class DiscountService {
    private final PremiumPolicy policy;
    public double getDiscount(User user) {
        return policy.isPremium(user) ? 0.15 : 0.0;
    }
}

public class ShippingService {
    private final PremiumPolicy policy;
    public boolean freeShipping(User user) {
        return policy.isPremium(user);
    }
}
```

Threshold muda → **um lugar**. Todos atualizam automaticamente.

---

DUPLICAÇÃO EM VALIDAÇÃO

```java
// ❌ validação espalhada
public class UserController {
    public ResponseEntity<?> create(@RequestBody UserRequest req) {
        if (req.getEmail() == null || !req.getEmail().contains("@"))
            return ResponseEntity.badRequest().body("Invalid email");
        // ...
    }
}

public class AdminController {
    public ResponseEntity<?> createAdmin(@RequestBody UserRequest req) {
        if (req.getEmail() == null || !req.getEmail().contains("@")) // cópia
            return ResponseEntity.badRequest().body("Invalid email");
        // ...
    }
}

// ✅ Bean Validation (conhecimento no modelo)
public class UserRequest {
    @Email
    @NotNull
    private String email;
}

// ambos controllers (zero duplicação)
public ResponseEntity<?> create(@Valid @RequestBody UserRequest req) { ... }
```

## TIPOS DE DUPLICAÇÃO

| Tipo | Exemplo | Solução |
| ---- | ------- | ------- |
| Código literal | Mesmo bloco copiado | Extrai método/classe |
| Conhecimento | Regra de negócio em N lugares | Centraliza em policy/service |
| Estrutura | Schema ≠ model ≠ DTO desalinhados | Geração automática / mapeamento |
| Documentação | Comentário repete o código | Remove comentário |
| Dados | Constante magic number repetida | Constante nomeada |

## Trade-offs

- Over-DRY: abstração prematura
- Under-DRY: caos de duplicação

Heurística: Duplicar primeiro, abrstrair quando doer.
