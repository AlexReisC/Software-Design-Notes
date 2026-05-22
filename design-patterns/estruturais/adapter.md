# Adapter

O Adapter é um padrão estrutural que permite que interfaces incompatíveis trabalhem juntas.

Ele atua como uma camada de tradução entre dois contratos diferentes.

## Problema que resolve

Você possui:
- Um cliente que espera uma interface A
- Um serviço existente (ou legado/externo) que expõe uma interface B

E essas interfaces são incompatíveis.

### Exemplo do problema

```java
public interface PaymentProcessor {
    void pay(double amount);
}
```

Mas você precisa usar um SDK externo:

```java
public class ExternalPaymentGateway {
    public void makeTransaction(double valueInCents) {
        System.out.println("Pagamento externo: " + valueInCents);
    }
}
```

Problemas:
- Assinaturas diferentes
- Unidade diferente (double vs centavos)
- Sem controle sobre o código externo

### Solução

Criar uma classe intermediária que:
- Implementa a interface esperada
- Traduz chamadas para o formato do serviço externo

## Cenários de uso

1. **Integração com sistemas externos**
   - APIs de pagamento (Stripe, Mercado Pago, etc.)
   - SDKs de terceiros
   - Serviços legados
2. **Migração de sistemas**
   - Substituir uma implementação antiga gradualmente
   - Manter compatibilidade com código existente
3. **Padronização de interfaces**
   - Criar um contrato interno consistente
   - Esconder diferenças de fornecedores
4. **Anti-corruption layer (DDD)**
   - Isolar seu domínio de modelos externos

## Quando NÃO usar

1. **Quando você controla ambas as interfaces**. Nesse caso, refatore diretamente.

2. **Quando a adaptação é trivial e não recorrente**. Um método utilitário pode ser suficiente.

3. **Quando começa a acumular lógica de negócio**. Adapter deve apenas **traduzir**, não decidir regras.

4. **Quando múltiplas adaptações complexas surgem**. Pode indicar necessidade de:
   - Facade
   - ou redesign arquitetural

## Exemplo de código (Java)

**Interface alvo**

```java
public interface PaymentProcessor {
    void pay(double amount);
}
```

**Classe incompatível**

```java
public class ExternalPaymentGateway {

    public void makeTransaction(double valueInCents) {
        System.out.println("Pagamento externo: " + valueInCents);
    }
}
```

**Adapter**

```java
public class PaymentAdapter implements PaymentProcessor {

    private final ExternalPaymentGateway gateway;

    public PaymentAdapter(ExternalPaymentGateway gateway) {
        this.gateway = gateway;
    }

    @Override
    public void pay(double amount) {
        double valueInCents = amount * 100;
        gateway.makeTransaction(valueInCents);
    }
}
```

**Uso**

```java
public class Main {

    public static void main(String[] args) {
        ExternalPaymentGateway gateway = new ExternalPaymentGateway();
        PaymentProcessor processor = new PaymentAdapter(gateway);

        processor.pay(50.0);
    }
}
```

### Versão mais realista

Cenário: múltiplos provedores externos

```java
@Component
public class StripeAdapter implements PaymentProcessor {

    private final StripeClient stripeClient;

    public StripeAdapter(StripeClient stripeClient) {
        this.stripeClient = stripeClient;
    }

    @Override
    public void pay(double amount) {
        stripeClient.charge(amount);
    }
}
```

Você pode combinar com:
- Strategy → escolher provider em runtime
- Factory → resolver adapter correto

## Trade-offs

**Vantagens**
- Baixo acoplamento
- Isolamento de dependências externas
- Facilita testes (mock do adapter)
- Protege domínio (DDD)

**Desvantagens**
- Aumento no número de classes
- Pode introduzir camadas desnecessárias
- Debug pode ficar mais indireto

## Relação com outros patterns

**Adapter vs Facade**
- Adapter → traduz interface
- Facade → simplifica interface

**Adapter vs Decorator**
- Adapter → muda interface
- Decorator → adiciona comportamento

**Adapter + Strategy (muito comum)**
- Strategy define qual comportamento
- Adapter define como integrar