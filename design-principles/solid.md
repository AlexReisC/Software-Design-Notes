# SOLID

Agrupamento de cinco principios que visam melhorar manutenabilidade, extensibilidade e testabilidade para sistemas orientados a objetos.

## Conceito
Cada princípio aborda um modo de falha específico no design:
- Rigidez → difícil de mudar
- Fragilidade → mudanças quebram partes não relacionadas
- Imobilidade → difícil de reutilizar

## S - SINGLE RESPONSIBILITY PRINCIPLE (SRP)

**Definição:** Classe tem **um motivo** pra mudar. Não "faz uma coisa", tem **um ator** responsável por ela. Ator = pessoa/papel que solicita mudança.

> "A class should have one, and only one, reason to change." — Robert C. Martin

### Exemplos

❌ RUIM - múltiplos atores
```java
public class UserService {

    // Ator 1: RH / regra de negócio
    public void createUser(UserRequest request) {
        validateEmail(request.getEmail());
        User user = toEntity(request);
        repository.save(user);
    }

    // Ator 2: DBA / persistência
    public void save(User user) {
        String sql = "INSERT INTO users (name, email) VALUES (?, ?)";
        jdbcTemplate.update(sql, user.getName(), user.getEmail());
    }

    // Ator 3: DevOps / infra de notificação
    public void sendWelcomeEmail(User user) {
        String body = "Bem vindo, " + user.getName();
        emailClient.send(user.getEmail(), "Bem vindo!", body);
    }

    // Ator 4: Segurança / auditoria
    public void logUserCreation(User user) {
        auditLog.write("USER_CREATED", user.getId(), LocalDateTime.now());
    }
}
```

Problema → mudança em email dispara recompilação de tudo. DBA altera SQL → testa /regra de negócio. 4 motivos pra mudar.

✅ LIMPO - um ator por classe
```java
// Ator: regra de negócio
public class UserService {
    private final UserRepository repository;
    private final EmailNotifier notifier;
    private final AuditLogger auditLogger;

    public void createUser(UserRequest request) {
        validateEmail(request.getEmail());
        User user = repository.save(toEntity(request));
        notifier.sendWelcome(user);
        auditLogger.logCreation(user);
    }

    private void validateEmail(String email) {
        if (!email.contains("@")) throw new InvalidEmailException(email);
    }
}

// Ator: persistência
public class UserRepository {
    public User save(User user) {
        String sql = "INSERT INTO users (name, email) VALUES (?, ?)";
        jdbcTemplate.update(sql, user.getName(), user.getEmail());
        return user;
    }
}

// Ator: notificação
public class EmailNotifier {
    public void sendWelcome(User user) {
        emailClient.send(user.getEmail(), "Bem vindo!", buildBody(user));
    }
}

// Ator: auditoria
public class AuditLogger {
    public void logCreation(User user) {
        auditLog.write("USER_CREATED", user.getId(), LocalDateTime.now());
    }
}
```

### COMO IDENTIFICAR VIOLAÇÃO

Pergunta: "Quem pede essa mudança?"
```markdown
"Precisa mudar o template do email"     → Marketing
"Precisa mudar a query do banco"        → DBA
"Precisa mudar a regra de validação"    → Product Owner
"Precisa mudar o formato do audit log"  → Segurança
```

Se respostas diferentes apontam pra mesma classe → violação SRP.

---

SRP != "uma função por classe"
```java
// ❌ Interpretação errada - classe anêmica
public class EmailValidator {
    public boolean validate(String email) {
        return email.contains("@");
    }
}

// ✅ SRP real - agrupa comportamentos do mesmo ator
public class UserValidator {
    public void validate(UserRequest request) {
        validateEmail(request.getEmail());
        validateAge(request.getAge());
        validateDocument(request.getCpf());
    }
    // todos respondem ao mesmo ator: regra de negócio de usuário
}
```

### Trade-offs
- Muito rigorosa → explosão de classes (muitas são criadas)
- Muito frouxo → classe deus (com muitas responsabilidades)

## O - OPEN/CLOSED PRINCIPLE (OCP)

**Definição:** Entidade aberta pra **extensão**, fechada pra **modificação**. Adiciona comportamento novo → cria código novo. Não altera código existente que já funciona.

> "Open for extension, closed for modification." — Bertrand Meyer

### Exemplos

❌ RUIM - modifica pra cada novo tipo

```java
public class PaymentProcessor {

    public void process(Payment payment) {
        if (payment.getType().equals("CREDIT_CARD")) {
            creditCardGateway.charge(payment.getAmount());

        } else if (payment.getType().equals("PIX")) {
            pixGateway.transfer(payment.getAmount(), payment.getKey());

        } else if (payment.getType().equals("BOLETO")) {
            boletoGateway.generate(payment.getAmount(), payment.getDueDate());

        // novo método? abre essa classe, adiciona else if
        // risco de quebrar CREDIT_CARD e PIX existentes
        }
    }
}
```
Problema → cada novo método de pagamento → modifica PaymentProcessor. Código testado volta ao risco.

✅ LIMPO - extensão via abstração
```java
// Contrato fechado - não muda
public interface PaymentStrategy {
    void process(Payment payment);
}

// Extensões - cada uma isolada
public class CreditCardPayment implements PaymentStrategy {
    public void process(Payment payment) {
        creditCardGateway.charge(payment.getAmount());
    }
}

public class PixPayment implements PaymentStrategy {
    public void process(Payment payment) {
        pixGateway.transfer(payment.getAmount(), payment.getKey());
    }
}

public class BoletoPayment implements PaymentStrategy {
    public void process(Payment payment) {
        boletoGateway.generate(payment.getAmount(), payment.getDueDate());
    }
}

// Novo método? Cria classe nova. PaymentProcessor não muda.
public class CryptoPayment implements PaymentStrategy {
    public void process(Payment payment) {
        cryptoGateway.send(payment.getAmount(), payment.getWalletAddress());
    }
}
```

```java
// Processor fechado pra modificação
public class PaymentProcessor {
    private final Map<String, PaymentStrategy> strategies;

    public PaymentProcessor(Map<String, PaymentStrategy> strategies) {
        this.strategies = strategies;
    }

    public void process(Payment payment) {
        PaymentStrategy strategy = strategies.get(payment.getType());
        if (strategy == null) throw new UnsupportedPaymentException(payment.getType());
        strategy.process(payment);
    }
}
```

```java
// Config Spring - novo método = novo Bean, zero alteração em classe existente
@Configuration
public class PaymentConfig {
    @Bean
    public Map<String, PaymentStrategy> paymentStrategies(
        CreditCardPayment cc, PixPayment pix, BoletoPayment boleto
    ) {
        return Map.of("CREDIT_CARD", cc, "PIX", pix, "BOLETO", boleto);
    }
}
```

### PADRÕES QUE IMPLEMENTAM OCP
| Padrão | Quando usar |
| ------ | ----------- |
| Strategy | Algoritmos/comportamentos intercambiáveis |
| Template | MethodFluxo fixo, passos variáveis |
| Decorator | Adiciona responsabilidade sem alterar classe |
| Chain of Responsibility | Pipeline de processamento extensível |

### Trade-offs
- Mais abstrações
- Difícil de navegar inicialmente

### OCP NA PRÁTICA - onde aplicar

```
✅ Aplica OCP:
→ Lógica varia por tipo (pagamento, notificação, exportação)
→ Regras de negócio mudam frequentemente
→ Plugin/extensão por terceiros

❌ Não over-engineer:
→ Lógica estável que raramente muda
→ CRUD simples sem variação de comportamento
```

## L - LISKOV SUBSTITUTION PRINCIPLE (LSP)

**Definição:** Subclasse substituí superclasse sem quebrar comportamento. Caller usa base → não sabe, não liga qual implementação concreta. Contrato da base = garantido pela filha.

> "Se S é subtipo de T, objetos de T podem ser substituídos por S sem alterar corretude do programa." — Barbara Liskov, 1987

### Exemplos de código

❌ RUIM - subclasse quebra contrato

```java
public class Rectangle {
    protected int width;
    protected int height;

    public void setWidth(int width)   { this.width = width; }
    public void setHeight(int height) { this.height = height; }

    public int area() { return width * height; }
}

public class Square extends Rectangle {

    // Quadrado força width == height - viola contrato de Rectangle
    @Override
    public void setWidth(int width) {
        this.width = width;
        this.height = width; // side effect inesperado
    }

    @Override
    public void setHeight(int height) {
        this.width = height;
    }
        this.height = height; // side effect inesperado
}
```

```java
// Caller confia no contrato de Rectangle
void resizeAndAssert(Rectangle r) {
    r.setWidth(5);
    r.setHeight(3);
    assert r.area() == 15; // ✅ Rectangle: passa
                           // ❌ Square: area() == 9 — QUEBRA
}
```

✅ LIMPO — abstração correta

```java
// Abstração que ambos respeitam
public interface Shape {
    int area();
}

public class Rectangle implements Shape {
    private final int width;
    private final int height;

    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }

    public int area() { return width * height; }
}

public class Square implements Shape {
    private final int side;

    public Square(int side) { this.side = side; }

    public int area() { return side * side; }
}

// Caller usa Shape — qualquer impl funciona corretamente
void printArea(Shape shape) {
    System.out.println("Area: " + shape.area()); // sempre correto
}
```

### SINAIS DE VIOLAÇÃO LSP

```java
// 🚨 Sinal 1 — instanceof no caller
public void process(Animal animal) {
    if (animal instanceof Dog dog) {
        dog.fetch();
    } else if (animal instanceof Cat cat) {
        cat.purr();
    }
    // caller precisa saber o tipo concreto → LSP violado
}

// 🚨 Sinal 2 — método lança exceção na subclasse
public class ReadOnlyRepository extends UserRepository {
    @Override
    public void save(User user) {
        throw new UnsupportedOperationException(); // quebra contrato
    }
}

// 🚨 Sinal 3 — pré-condição mais restritiva na subclasse
public class PositiveCalculator extends Calculator {
    @Override
    public int divide(int a, int b) {
        if (a < 0 || b < 0) throw new IllegalArgumentException(); // base não tinha essa restrição
        return a / b;
    }
}
```

### Trade-offs
- Evite herança "ingenua"
- Prefira interfaces

## I - INTERFACE SEGREGATION PRINCIPLE (ISP)

**Definição:** Cliente não deve depender de métodos que não usa. Interface cheia → quebra em interfaces menores e específicas.

> "Many client-specific interfaces are better than one general-purpose interface." — Robert C. Martin

### Exemplos de código

❌ RUIM - interface gorda
```java
public interface UserRepository {
    User findById(UUID id);
    List<User> findAll();
    void save(User user);
    void delete(UUID id);
    void update(User user);
    List<User> findByAuditLog();      // só auditoria usa
    void exportToCsv(String path);    // só relatório usa
    void syncWithLdap();              // só admin usa
}
```
```java
// Só precisa ler - forçado a implementar save, delete, sync...
public class ReadOnlyUserService {
    private final UserRepository repository; // carrega métodos inúteis

    public User getUser(UUID id) {
        return repository.findById(id);
    }
}
```
Problema → `ReadOnlyUserService` acoplado a `syncWithLdap()`. Muda assinatura → recompila quem nem usa.

✅ LIMPO - interfaces segregadas

```java
// Cada interface = um papel
public interface UserReader {
    User findById(UUID id);
    List<User> findAll();
}

public interface UserWriter {
    void save(User user);
    void update(User user);
    void delete(UUID id);
}

public interface UserAuditor {
    List<User> findByAuditLog();
}

public interface UserExporter {
    void exportToCsv(String path);
}
```
```java
// Impl concreta implementa o que faz sentido
public class UserRepositoryImpl
    implements UserReader, UserWriter, UserAuditor, UserExporter {
    // implementa tudo
}

// Cada cliente depende só do que usa
public class ReadOnlyUserService {
    private final UserReader reader; // só isso

    public User getUser(UUID id) { return reader.findById(id); }
}

public class AdminService {
    private final UserWriter writer;
    private final UserAuditor auditor; // só isso
}

public class ReportService {
    private final UserExporter exporter; // só isso
}
```

### SINAIS DE VIOLAÇÃO

```java
// 🚨 Impl joga exceção pra método que não faz sentido
public class ReportRepository implements UserRepository {
    public void syncWithLdap() {
        throw new UnsupportedOperationException(); // LSP violado também
    }
}

// 🚨 Método sempre retorna null/vazio na impl
public class CacheRepository implements UserRepository {
    public void exportToCsv(String path) { /* não faz nada */ }
}
```
Método vazio ou exception → interface gorda → ISP violado.

### Trade-offs
- Muitas interfaces
- Melhor flexibilidade

## D - DEPENDENCY INVERSION PRINCIPLE (DIP)

**Definição:** Módulo alto nível não depende de módulo baixo nível. Ambos dependem de abstração. Abstração não depende de detalhe. Detalhe depende de abstração.

> "High-level modules should not depend on low-level modules. Both should depend on abstractions." — Robert C. Martin

NIVEÍS
```
Alto nível  → regra de negócio (UserService)
Baixo nível → detalhe técnico (JdbcUserRepository, SmtpEmailSender)
```

Sem DIP → alto nível conhece e instancia baixo nível. Acoplamento direto.

### Exemplos de código

❌ RUIM — alto nível instancia baixo nível

```java
public class UserService {

    // instancia direto — acoplamento concreto
    private final JdbcUserRepository repository = new JdbcUserRepository();
    private final SmtpEmailSender emailSender = new SmtpEmailSender();

    public void createUser(UserRequest request) {
        User user = repository.save(toEntity(request));
        emailSender.sendWelcome(user.getEmail());
    }
}
```

Problema:
- Troca Jdbc por MongoDB → modifica UserService
- Troca Smtp por SES → modifica UserService
- Teste unitário → obrigado a subir JDBC + SMTP
- Alto nível acoplado a detalhe técnico

✅ LIMPO — ambos dependem de abstração

```java
// Abstrações — contratos
public interface UserRepository {
    User save(User user);
    Optional<User> findById(UUID id);
}

public interface EmailSender {
    void sendWelcome(String email);
}

// Alto nível — depende só de abstração
public class UserService {
    private final UserRepository repository;
    private final EmailSender emailSender;

    // abstração injetada — não instancia nada
    public UserService(UserRepository repository, EmailSender emailSender) {
        this.repository = repository;
        this.emailSender = emailSender;
    }

    public void createUser(UserRequest request) {
        User user = repository.save(toEntity(request));
        emailSender.sendWelcome(user.getEmail());
    }
}

// Baixo nível — depende da abstração (implementa)
public class JdbcUserRepository implements UserRepository {
    public User save(User user) { /* jdbc impl */ }
    public Optional<User> findById(UUID id) { /* jdbc impl */ }
}

public class MongoUserRepository implements UserRepository {
    public User save(User user) { /* mongo impl */ }
    public Optional<User> findById(UUID id) { /* mongo impl */ }
}

public class SmtpEmailSender implements EmailSender {
    public void sendWelcome(String email) { /* smtp impl */ }
}

public class SesEmailSender implements EmailSender {
    public void sendWelcome(String email) { /* aws ses impl */ }
}
```
```java
// Spring gerencia — UserService não sabe qual impl concreta
@Configuration
public class AppConfig {
    @Bean
    public UserRepository userRepository() {
        return new MongoUserRepository(); // troca aqui, zero impacto em UserService
    }

    @Bean
    public EmailSender emailSender() {
        return new SesEmailSender();
    }
}
```

### Trade-offs
- Requer injeção de dependência
- Setup mais complexo

### DIP vs DI vs IoC
```
DIP → princípio: depende de abstração
DI  → padrão: injeta dependência de fora
IoC → mecanismo: container gerencia criação (Spring)

DIP é o porquê.
DI é o como.
IoC é quem faz.
```

## SOLID - RESUMO COMPLETO

| Princípio | Core | Sinal de violação |
| --------- | ---- | ----------------- |
| SRP | Um motivo pra mudar | Classe com múltiplos atores |
| OCP | Estende sem modificar | `if/else` cresce pra novo tipo |
| LSP | Subtipo honra contrato | `instanceof` no caller |
| ISP | Interface enxuta | Impl com método vazio/exception |
| DIP | Depende de abstração | `new ConcreteClass()` no alto nível |

## Como SOLID trabalha junto

| Princípio | Foco | Previne |
| --------- | ---- | ------- |
| SRP | Responsabilidade | God classes |
| OCP | Extensibilidade | Mudanças frequentes no código |
| LSP | Herança correta | Quebra no polimorfismo |
| ISP | Design de interface | Interfaces cheias |
| DIP | Dependency direction | Alto acoplamento |

## Insigths
- SOLID é sobre controlar a mudança
- Uso excessivo de SOLID → complexidade desnecessária
- Comece simples (KISS, YAGNI)
- Prefira composição + interfaces