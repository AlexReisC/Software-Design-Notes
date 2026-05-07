# Strategy

**Definição:** O Strategy é um padrão comportamental que define uma família de algoritmos, encapsula cada um deles e os torna intercambiáveis em tempo de execução.

Ele permite que o comportamento de um objeto varie sem alterar sua estrutura interna.

## Problema que resolve

Evita:
- **if/else** ou **switch extensivos** baseados em tipo/comportamento
- Violação do **Open/Closed Principle (OCP)**: necessidade de modificar código existente para adicionar novos comportamentos

### Exemplo do problema

```java
public class PaymentService {

    public void processPayment(String type, double amount) {
        if (type.equals("CREDIT_CARD")) {
            // lógica cartão
        } else if (type.equals("PIX")) {
            // lógica pix
        } else if (type.equals("BOLETO")) {
            // lógica boleto
        }
    }
}
```

Problemas:
- Crescimento exponencial de condicionais
- Baixa coesão
- Alto acoplamento
- Difícil de testar isoladamente

### Solução (Strategy)

Extrair cada comportamento para uma estratégia independente:
- Interface comum
- Implementações concretas
- Contexto que delega execução

## Cenários de uso

Use Strategy quando:

1. Múltiplos algoritmos intercambiáveis
- Cálculo de frete (SEDEX, PAC, transportadora)
- Formas de pagamento
- Algoritmos de ordenação
2. Regras que mudam frequentemente
- Regras de desconto
- Validações condicionais
3. Evitar condicionais complexas
- Substituir `switch` por polimorfismo
4. Configuração em runtime
- Seleção dinâmica baseada em:
  - banco de dados
  - feature flags
  - contexto da requisição

## Quando NÃO USAR

Evite Strategy quando:

1. **Poucas variações simples**
- Se existem apenas 2 variações triviais, o overhead não compensa.
2. **Algoritmos não variam**
- Se não há expectativa de mudança/extensão → overengineering
3. **Alto número de estratégias pequenas**
- Pode gerar:
  - excesso de classes
  - dificuldade de navegação
4. **Dependência forte de estado compartilhado**

Strategies devem ser preferencialmente stateless
Se dependem fortemente do contexto → possível smell

## Exemplos de código

**Interface Strategy**

```java
public interface PaymentStrategy {
    void pay(double amount);
}
```

**Implementações concretas**

```java
public class CreditCardPayment implements PaymentStrategy {

    @Override
    public void pay(double amount) {
        System.out.println("Pagamento via cartão: " + amount);
    }
}
```

```java
public class PixPayment implements PaymentStrategy {

    @Override
    public void pay(double amount) {
        System.out.println("Pagamento via PIX: " + amount);
    }
}
```

**Contexto**

```java
public class PaymentContext {

    private PaymentStrategy strategy;

    public PaymentContext(PaymentStrategy strategy) {
        this.strategy = strategy;
    }

    public void setStrategy(PaymentStrategy strategy) {
        this.strategy = strategy;
    }

    public void execute(double amount) {
        strategy.pay(amount);
    }
}
```

**Uso**

```java
public class Main {
    public static void main(String[] args) {

        PaymentContext context = new PaymentContext(new PixPayment());
        context.execute(100);

        context.setStrategy(new CreditCardPayment());
        context.execute(200);
    }
}
```
### Versão mais realista (Spring Boot)

Evita if usando injeção de dependência + map de estratégias:

```java
@Component
public class PaymentStrategyFactory {

    private final Map<String, PaymentStrategy> strategies;

    public PaymentStrategyFactory(List<PaymentStrategy> strategyList) {
        this.strategies = strategyList.stream()
                .collect(Collectors.toMap(s -> s.getClass().getSimpleName(), s -> s));
    }

    public PaymentStrategy get(String type) {
        return strategies.get(type);
    }
}
```

Ou mais robusto:

```java
public interface PaymentStrategy {
    String getType();
    void pay(double amount);
}
```

## Trade-offs

**Vantagens**
- Baixo acoplamento
- Alta coesão
- Extensível (OCP)
- Testável

**Desvantagens**
- Mais classes
- Pode aumentar complexidade estrutural
- Requer disciplina de organização

## Relação com outros patterns

- Factory → pode ser usada para instanciar a Strategy
- State → semelhante estruturalmente, mas muda comportamento baseado em estado interno
- Command → encapsula ações, não algoritmos intercambiáveis