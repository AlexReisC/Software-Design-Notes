# SINGLETON

**Definição**: Garante **uma única instância** de classe no sistema. Ponto de acesso global controlado.

## Ideia central

Controlar:
- quantidade de instâncias
- ciclo de vida
- acesso global

Estrutura conceitual:

```java
Cliente
   ↓
Singleton.getInstance()
   ↓
Instância única
```

## Quando usar

- Configuração global (lida uma vez, lida em todo lugar)
- Recursos caros (objetos pesados, compartilháveis, centralizados)
- Logger (instância única faz sentido)
- Cache compartilhado
- Registry/metadata (mapeamentos globais)

## Evita quando

- Estado global excessivo (Singleton frequentemente vira variável global disfarçada)
  - acoplamento oculto
  - dependências implícitas
- Precisa de mais de uma instância no futuro
- Dificulta teste (estado global = teste acoplado)
- Viola SRP (controla própria criação E faz trabalho)

## Implementação clássica (thread-safe)

```java
public class DatabaseConnectionPool {
    // volatile garante visibilidade entre threads
    private static volatile DatabaseConnectionPool instance;
    private final List<Connection> pool;

    // construtor privado - bloqueia instanciação externa
    private DatabaseConnectionPool() {
        pool = initializePool();
    }

    // double-checked locking - thread-safe + performático
    public static DatabaseConnectionPool getInstance() {
        if (instance == null) {                          // check 1 - sem lock (rápido)
            synchronized (DatabaseConnectionPool.class) {
                if (instance == null) {                  // check 2 - com lock (seguro)
                    instance = new DatabaseConnectionPool();
                }
            }
        }
        return instance;
    }

    public Connection acquire() { ... }
    public void release(Connection conn) { ... }
}
```

## Implementação mordena - Enum

```java
// Enum Singleton - thread-safe por spec JVM, serialização gratuita
public enum AppConfig {
    INSTANCE;

    private final Properties properties;

    AppConfig() {
        properties = new Properties();
        try {
            properties.load(
                getClass().getResourceAsStream("/application.properties")
            );
        } catch (IOException e) {
            throw new RuntimeException("Failed to load config", e);
        }
    }

    public String get(String key) {
        return properties.getProperty(key);
    }

    public String get(String key, String defaultValue) {
        return properties.getProperty(key, defaultValue);
    }
}

// Uso
String dbUrl = AppConfig.INSTANCE.get("database.url");
```

### Vantagens do enum Singleton
- thread-safe automaticamente
- simples
- protegido contra reflection
- protegido contra serialização


## Implementação - Initialization-on-demand (lazy + thread-safe)
```java
public class CacheManager {
    private final Map<String, Object> cache = new ConcurrentHashMap<>();

    // JVM garante: classe interna carregada só quando referenciada
    // thread-safe sem synchronized
    private static class Holder {
        static final CacheManager INSTANCE = new CacheManager();
    }

    private CacheManager() {}

    public static CacheManager getInstance() {
        return Holder.INSTANCE; // lazy + thread-safe
    }

    public void put(String key, Object value) { cache.put(key, value); }
    public Optional<Object> get(String key)   { return Optional.ofNullable(cache.get(key)); }
    public void evict(String key)             { cache.remove(key); }
}
```

## Singleton no Spring

Por padrão:

```java
@Component
@Service
@Repository
```

são **singleton scoped**. Uma instância por ApplicationContext.

### Singleton no Spring - @Bean padrão

```java
// Spring gerencia Singleton por padrão
// Não precisa implementar padrão manualmente

@Service // scope = singleton por padrão
public class UserService {
    // Spring cria uma instância, injeta em todos que dependem
}

// Explícito
@Bean
@Scope("singleton") // default - redundante mas válido
public PaymentGateway paymentGateway() {
    return new StripeGateway(apiKey);
}

// Prototype - nova instância a cada injeção (oposto de Singleton)
@Bean
@Scope("prototype")
public ReportBuilder reportBuilder() {
    return new ReportBuilder();
}
```

## Singleton vs Static Class

```java
// Static - sem instância, sem polimorfismo, sem injeção
public class MathUtils {
    public static double calculateTax(double amount) { return amount * 0.1; }
}
// Problema: não mockável, não substituível
```

```java
// Singleton - instância, polimorfismo possível, injetável
public interface TaxCalculator {
    double calculate(double amount);
}

public enum StandardTaxCalculator implements TaxCalculator {
    INSTANCE;
    public double calculate(double amount) { return amount * 0.1; }
}
// Mockável, substituível, testável
```

Singleton possui:
- objeto real
- estado
- polimorfismo possível
- interfaces

## Arquitetura moderna

Singleton perdeu espaço com:
- Spring
- CDI
- Guice
- containers DI

Hoje ele aparece mais como:
- lifecycle scope
- singleton bean
- resource manager

do que como implementação manual clássica.

## Trade-offs

**Vantagens**:
- Controle de instância: Evita múltiplas criações.
- Compartilhamento eficiente: Economiza recursos.
- Ponto global de acesso: Fácil acesso centralizado.

**Desvantagens**
- Alto acoplamento: Código depende de estado global.
- Testabilidade ruim: Dificulta mocking.
- Concorrência complexa: Especialmente se stateful.
- Dependências ocultas: Objetos podem acessar singleton implicitamente.

