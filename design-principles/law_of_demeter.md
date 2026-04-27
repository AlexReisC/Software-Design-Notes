# LAW OF DEMETER - PRINCÍPIO DO MENOR CONHECIMENTO

**Definição:** Módulo conhece só seus vizinhos imediatos. Não fala com estranhos. Método chama só o que está na própria classe ou foi diretamente injetado/passado.

## Conceito

Evite encadear vairas chamadas em vários objetos.

## Regra formal

Método `m` de objeto `O` pode chamar métodos de:
```
1. O mesmo (this)
2. Parâmetros de m
3. Objetos criados dentro de m
4. Dependências diretas de O (campos)

❌ Nunca: objeto retornado por outro objeto
```

## Exemplos de código

❌ RUIM - train wreck / cadeia de chamadas

```java
// Acessa entranhas de entranhas de entranhas
public class OrderService {

    public String getCustomerCity(Order order) {
        return order.getCustomer()        // Order conhece Customer ✅
                    .getAddress()         // Customer conhece Address ✅
                    .getCity()            // Address conhece City ✅
                    .getName();           // mas OrderService conhece TUDO ❌
    }

    public double calculateShipping(Order order) {
        String country = order.getCustomer()
                              .getAddress()
                              .getCountry()
                              .getCode();  // 4 níveis de profundidade ❌

        return shippingTable.getRate(country) * order.getWeight();
    }
}
```

Problema:
- OrderService depende de Order, Customer, Address, City, Country
- Muda estrutura de Address → OrderService quebra
- Acoplamento estrutural profundo

✅ BOM - delega pra vizinho imediato

```java
// Cada classe expõe o que o caller precisa, esconde estrutura interna
public class Order {
    private Customer customer;

    public String getCustomerCity() {
        return customer.getCity(); // Order delega pra Customer
    }

    public String getCustomerCountryCode() {
        return customer.getCountryCode();
    }
}

public class Customer {
    private Address address;

    public String getCity() {
        return address.getCityName(); // Customer delega pra Address
    }

    public String getCountryCode() {
        return address.getCountryCode();
    }
}

public class Address {
    private City city;
    private Country country;

    public String getCityName()    { return city.getName(); }
    public String getCountryCode() { return country.getCode(); }
}
```

---

EXCEÇÃO LEGÍTIMA - fluent API / builder

```java
// ❌ LoD violation?
User user = User.builder()
    .name("Alex")
    .email("alex@example.com")
    .role(Role.ADMIN)
    .build();

// ✅ NÃO é violation — builder retorna sempre o mesmo objeto (si mesmo)
// Cada chamada é no mesmo builder, não em objetos diferentes
```
```java
// ❌ LoD violation real disfarçado de fluent
order.getPayment().getCard().getIssuer().validate(); // objetos diferentes
```

Regra: **mesmo objeto** → fluent OK. **Objetos diferentes** → LoD violation.

## LoD EM SPRING

```java
// ❌ Controller acessa estrutura interna de Service
@GetMapping("/orders/{id}/city")
public String getCity(@PathVariable UUID id) {
    return orderService.findById(id)  // retorna Order
                       .getCustomer() // Controller conhece Customer
                       .getAddress()  // Controller conhece Address
                       .getCity()     // Controller conhece City
                       .getName();    // 4 níveis ❌
}

// ✅ Controller pede resultado — não navega estrutura
@GetMapping("/orders/{id}/city")
public String getCity(@PathVariable UUID id) {
    return orderService.getCustomerCity(id); // um nível ✅
}

// Service encapsula navegação
public class OrderService {
    public String getCustomerCity(UUID orderId) {
        Order order = repository.findById(orderId).orElseThrow();
        return order.getCustomerCity();
    }
}
```

## CUSTO DE IGNORAR O LoD

```java
Muda Address.city pra Address.location.cityName
→ quebra OrderService, ShippingService, ReportService, InvoiceService
→ grep no codebase inteiro
→ 40 arquivos alterados
→ regression test em tudo
```
```java
Com LoD:
→ muda Address internamente
→ atualiza getCityName() em Address
→ zero impacto externo
→ 1 arquivo alterado
```
