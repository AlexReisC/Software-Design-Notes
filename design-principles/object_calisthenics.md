# Object Calisthenics

**Definição:** 9 regras de Jeff Bay para forçar OOP bem feito. Restrições intencionais que eliminam maus hábitos. Seguir todas → código naturalmente coeso, encapsulado, testável.

> "Rules are meant to be broken — but understand them first." — Jeff Bay

## REGRA 1: Um nível de identação

**Definição:** Método tem só 1 nível de `{}` aninhado. Força extração de método.

```java
// ❌ múltiplos níveis
public void processOrders(List<Order> orders) {
    for (Order order : orders) {                    // nível 1
        if (order.isPending()) {                    // nível 2
            for (Item item : order.getItems()) {   // nível 3
                if (item.isAvailable()) {          // nível 4
                    process(item);
                }
            }
        }
    }
}

// ✅ um nível - extrai método
public void processOrders(List<Order> orders) {
    orders.forEach(this::processPendingOrder);    // nível 1
}

private void processPendingOrder(Order order) {
    if (!order.isPending()) return;
    order.getItems().forEach(this::processIfAvailable); // nível 1
}

private void processIfAvailable(Item item) {
    if (item.isAvailable()) process(item);        // nível 1
}
```

## REGRA 2: NÃO USE ELSE

**Definição:** Elimina else. Usa early return, polimorfismo, ou guard clause.

```java
// ❌ else encadeado
public double calculateDiscount(User user) {
    if (user.isPremium()) {
        if (user.isVip()) {
            return 0.30;
        } else {
            return 0.15;
        }
    } else {
        return 0.0;
    }
}

// ✅ early return (sem else)
public double calculateDiscount(User user) {
    if (user.isVip())     return 0.30;
    if (user.isPremium()) return 0.15;
    return 0.0;
}

// ✅✅ polimorfismo (sem if/else algum)
public interface DiscountPolicy {
    double calculate();
}

public class VipDiscount     implements DiscountPolicy { public double calculate() { return 0.30; } }
public class PremiumDiscount implements DiscountPolicy { public double calculate() { return 0.15; } }
public class NoDiscount      implements DiscountPolicy { public double calculate() { return 0.0;  } }
```

## REGRA 3: ENVOLVA PRIMITIVOS

**Definição:** Primitivo com comportamento/regra → envolve em classe. Evita Primitive Obsession.

```java
// ❌ primitivo nu: sem regra, sem semântica
public class Order {
    private double amount;  // pode ser negativo? qual moeda?
    private String email;   // válido? formato?
    private String cpf;     // válido? com máscara? sem?
}

void transfer(double amount, String accountId) { ... } // o que é amount? BRL? USD?
```

```java
// ✅ primitivo envolto: regra e semântica no tipo

public class Money {
    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        if (amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Amount must be >= 0");
        this.amount   = amount;
        this.currency = currency;
    }

    public Money add(Money other) {
        if (!currency.equals(other.currency))
            throw new CurrencyMismatchException();
        return new Money(amount.add(other.amount), currency);
    }

    public boolean isGreaterThan(Money other) {
        return amount.compareTo(other.amount) > 0;
    }
}

public class Email {
    private final String value;

    public Email(String value) {
        if (value == null || !value.contains("@"))
            throw new InvalidEmailException(value);
        this.value = value.toLowerCase().trim();
    }

    public String getValue() { return value; }
}

public class Cpf {
    private final String digits; // armazena só dígitos

    public Cpf(String raw) {
        String digits = raw.replaceAll("\\D", "");
        if (!isValid(digits))
            throw new InvalidCpfException(raw);
        this.digits = digits;
    }

    public String formatted() {
        return "%s.%s.%s-%s".formatted(
            digits.substring(0,3), digits.substring(3,6),
            digits.substring(6,9), digits.substring(9,11)
        );
    }
}

// Order - tipos expressivos, regras embutidas
public class Order {
    private final Money amount;
    private final Email customerEmail;
    private final Cpf customerCpf;
}
```

## REGRA 4: COLEÇÕES DE PRIMEIRA CLASSE

**Definição:** Classe que contém coleção não contém outros campos. Coleção + comportamento = classe própria.

```java
// ❌ coleção nua - comportamento espalhado nos callers
public class Order {
    private List<Item> items;
    private String status;
    private double total;
}

// Callers duplicam lógica
double total = order.getItems().stream()
    .mapToDouble(i -> i.getPrice() * i.getQuantity()).sum(); // OrderService
double total = order.getItems().stream()
    .mapToDouble(i -> i.getPrice() * i.getQuantity()).sum(); // InvoiceService - duplicado
```

```java
// ✅ coleção de primeira classe - comportamento encapsulado
public class OrderItems {
    private final List<Item> items;

    public OrderItems(List<Item> items) {
        if (items == null || items.isEmpty())
            throw new InvalidOrderException("Order must have items");
        this.items = List.copyOf(items); // imutável
    }

    public Money total() {
        return items.stream()
            .map(Item::subtotal)
            .reduce(Money.ZERO, Money::add);
    }

    public boolean contains(Product product) {
        return items.stream().anyMatch(i -> i.getProduct().equals(product));
    }

    public OrderItems applyDiscount(DiscountPolicy policy) {
        return new OrderItems(items.stream()
            .map(policy::apply)
            .toList());
    }

    public int count() { return items.size(); }
}

// Order - usa coleção tipada
public class Order {
    private final OrderItems items; // não tem List<Item> - tem OrderItems
    private OrderStatus status;

    public Money total() { return items.total(); } // delega
}
```

## REGRA 5: UM PONTO POR LINHA

Não encadeia chamadas em objetos diferentes.

```java
// ❌
order.getCustomer().getAddress().getCity().getName();

// ✅
order.getCustomerCity();
```

## REGRA 6: NÃO ABREVIE NOMES

**Definição:** Nome completo sempre. Abreviação = contexto perdido.

```java
// ❌
int d;
String usr;
void calcTot(List<Itm> itms) { ... }
class UsrMgr { ... }

// ✅
int elapsedDays;
String username;
void calculateTotal(List<Item> items) { ... }
class UserManager { ... }

// Exceção legítima - convenção universal
for (int i = 0; i < size; i++) // i ok em loop trivial
Map<K, V>                       // K, V ok em generics
```

## REGRA 7: ENTIDADES PEQUENAS

```
Classe   → máx 50 linhas
Pacote   → máx 10 arquivos
```

```java
// ❌ classe com 300 linhas: faz demais
public class UserService { // 300 linhas
    // autenticação + CRUD + notificação + relatório + permissão
}

// ✅ classes pequenas, coesas
public class UserAuthService    { } // ~40 linhas
public class UserCrudService    { } // ~45 linhas
public class UserNotifier       { } // ~30 linhas
public class UserReportService  { } // ~35 linhas
```

## REGRA 8: MÁX 2 VARIÁVEIS DE INSTÂNCIA

**Definição:** Restrição extrema. Força coesão máxima e separação de conceitos.

```java
// ❌ muitos campos - baixa coesão
public class User {
    private String name;
    private String email;
    private String street;
    private String city;
    private String country;
    private String phone;
}

// ✅ agrupa em tipos - cada classe com até 2 campos
public class Name {
    private final String first; // 1
    private final String last;  // 2
}

public class Address {
    private final Street street; // 1
    private final City city;     // 2
}

public class City {
    private final String name;    // 1
    private final Country country;// 2
}

public class User {
    private final Name name;      // 1
    private final Email email;    // 2
    // address? → UserProfile extends User ou composição
}
```
Regra 8 é a mais extrema. Em projetos reais → relaxa pra 3–5 campos. Espírito: agrupa campos relacionados em tipos próprios.

## REGRA 9: SEM GETTERS/SETTERS PÚBLICOS

**Definição:** Não expõe estado bruto. Objeto executa comportamento, não entrega dado pra caller decidir.

```java
// ❌ getter/setter livre - Tell Don't Ask violado
public class BankAccount {
    private double balance;

    public double getBalance() { return balance; }       // expõe dado
    public void setBalance(double balance) {             // setter livre
        this.balance = balance;
    }
}

// caller decide pelo objeto
if (account.getBalance() > amount) {
    account.setBalance(account.getBalance() - amount); // lógica fora do objeto
}

// ✅ comportamento - sem setter, sem getter de estado mutável
public class BankAccount {
    private double balance;

    public void withdraw(double amount) {               // Tell
        if (amount > balance) throw new InsufficientFundsException();
        balance -= amount;
    }

    public void deposit(double amount) {                // Tell
        if (amount <= 0) throw new IllegalArgumentException();
        balance += amount;
    }

    // getter ok pra leitura/display - não pra decisão
    public boolean hasSufficientFunds(double amount) { // pergunta semântica
        return balance >= amount;
    }

    public String formattedBalance() {                  // apresentação
        return "R$ %.2f".formatted(balance);
    }
}
```