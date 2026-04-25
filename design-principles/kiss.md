# KISS - KEEP IT SIMPLE, STUPID

**Definição:** Solução mais simples que resolve o problema = solução correta. Complexidade sem necessidade = dívida técnica voluntária.

## Conceito

Complexidade aumenta:
- carga cognitiva
- custo de manutenção

Simplicidade melhora:
- legibilidade
- previsibilidade
- mutabilidade

## Exemplos de código

❌ RUIM - over-engineered sem necessidade

```java
// Requisito: somar lista de números
// Dev resolve com Strategy + Factory + Builder

public interface SummationStrategy {
    double execute(List<Double> numbers);
}

public class IterativeSummation implements SummationStrategy {
    public double execute(List<Double> numbers) {
        double sum = 0;
        for (Double n : numbers) sum += n;
        return sum;
    }
}

public class SummationStrategyFactory {
    public static SummationStrategy create(String type) {
        return switch (type) {
            case "ITERATIVE" -> new IterativeSummation();
            default -> throw new IllegalArgumentException();
        };
    }
}

public class SummationExecutorBuilder {
    private String strategyType = "ITERATIVE";
    public SummationExecutorBuilder withStrategy(String type) {
        this.strategyType = type;
        return this;
    }
    public double execute(List<Double> numbers) {
        return SummationStrategyFactory.create(strategyType).execute(numbers);
    }
}

// uso
double result = new SummationExecutorBuilder()
    .withStrategy("ITERATIVE")
    .execute(numbers);
```

✅ KISS
```java
double result = numbers.stream().mapToDouble(Double::doubleValue).sum();
```

6 classes vs 1 linha. Mesmo resultado.

---

❌ RUIM - lógica condicional desnecessária

```java
public boolean isEven(int number) {
    if (number % 2 == 0) {
        return true;
    } else {
        return false;
    }
}
```

✅ KISS

```java
public boolean isEven(int number) {
    return number % 2 == 0;
}
```
---

❌ RUIM - cache prematuro sem evidência de problema

```java
// Requisito: buscar usuário por ID
// Dev adiciona cache multicamada "por precaução"
public class UserService {
    private final Map<UUID, User> l1Cache = new ConcurrentHashMap<>();
    private final RedisTemplate<UUID, User> l2Cache;
    private final UserRepository repository;

    public User findById(UUID id) {
        // L1
        User cached = l1Cache.get(id);
        if (cached != null) return cached;

        // L2
        User redisCached = l2Cache.opsForValue().get(id);
        if (redisCached != null) {
            l1Cache.put(id, redisCached);
            return redisCached;
        }

        // DB
        User user = repository.findById(id).orElseThrow();
        l2Cache.opsForValue().set(id, user, 10, TimeUnit.MINUTES);
        l1Cache.put(id, user);
        return user;
    }
}
```

✅ KISS — resolve o problema. Otimiza com dados reais.

```java
public User findById(UUID id) {
    return repository.findById(id).orElseThrow();
}
```

Cache só quando **profiler prova** que é necessário.

## KISS NÃO É CÓDIGO BURRO

```java
// ❌ "simples" mas ilegível
public double calc(List<Item> i) {
    double t = 0; for(Item x:i) t+=x.p*x.q; return t>100?t*0.9:t;
}

// ✅ KISS real — simples + legível
public double calculateTotal(List<Item> items) {
    double subtotal = items.stream()
        .mapToDouble(item -> item.getPrice() * item.getQuantity())
        .sum();
    return applyBulkDiscount(subtotal);
}

private double applyBulkDiscount(double subtotal) {
    return subtotal > 100 ? subtotal * 0.9 : subtotal;
}
```

KISS = menor complexidade acidental. Não menor número de linhas.

## COMPLEXIDADE ACIDENTAL vs ESSENCIAL

```
Essencial   → complexidade do problema em si
              (ex: cálculo de juros compostos é complexo por natureza)

Acidental   → complexidade que o dev introduz
              (ex: Factory pra somar lista, cache sem evidência)

KISS elimina acidental. Não toca essencial.
```

## SINAIS DE VIOLAÇÃO

| Sinal | Diagnóstico |
| ----- | ----------- |
| Padrão de design sem problema real | Over-engineering |
| Abstração com uma única impl | Prematura |
| Dev precisa de diagrama pra explicar função simples | Complexidade acidental |

Heurística: Otimize para os requisitos de hoje, não para os hipotéticos