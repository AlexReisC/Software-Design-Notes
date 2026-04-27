# YAGNI - YOU AIN'T GONNA NEED IT

**Definição:** Não implementa funcionalidade até ter necessidade **real e presente**. Código especulativo = dívida sem credor.

## Conceito

Design especulativo leva a:
- Código morto
- Abstrações erradas
- Esforço desperdiçado

## Exemplos de código

❌ RUIM - implementa o futuro imaginado

```java
// Requisito atual: sistema tem um tipo de usuário - CUSTOMER
public class User {
    private String name;
    private String email;

    // "pode ser útil no futuro"
    private UserType type;          // só existe CUSTOMER
    private String tenantId;        // multi-tenant não pedido
    private String preferredLang;   // i18n não pedido
    private String timezone;        // não pedido
    private boolean mfaEnabled;     // não pedido
    private String oauthProvider;   // não pedido
    private Map<String, Object> metadata; // "flexibilidade futura"
}
```
```java
// Service com suporte a "futuro"
public class UserService {
    public User createUser(UserRequest request) {
        validateForAllFutureUseCases(request); // valida coisas que não existem
        User user = toEntity(request);
        enrichWithDefaults(user);              // popula campos que ninguém usa
        notifyAllPossibleSystems(user);        // integra com sistemas hipotéticos
        return repository.save(user);
    }

    // método que ninguém chama
    public List<User> findByTenant(String tenantId) { ... }
    public void migrateToNewSchema(User user) { ... }
    public Report generateAuditReport(DateRange range) { ... }
}
```

✅ BOM - só o que o requisito pede

```java
// Requisito: criar usuário com nome e email
public class User {
    private String name;
    private String email;
    // só isso. Adiciona quando precisar.
}

public class UserService {
    public User createUser(UserRequest request) {
        validateEmail(request.getEmail());
        return repository.save(toEntity(request));
    }
}
```
Multi-tenant chega → adiciona `tenantId`. MFA chega → adiciona `mfaEnabled`. **Baseado em necessidade real, não suposição**.

## YAGNI EM ARQUITETURA

```java
// ❌ microservices desde o dia 1 "pra escalar no futuro"
UserService          → porta 8081
AuthService          → porta 8082
NotificationService  → porta 8083
PaymentService       → porta 8084
ReportService        → porta 8085
// 10 usuários em prod. Kubernetes. Service mesh. Circuit breaker.
// Complexidade de Netflix pra problema de startup.

// ✅ YAGNI - monólito modular primeiro
// Módulos bem separados dentro de um app.
// Quando escala real aparecer → extrai serviço com dados reais de bottleneck.
com.app.user
com.app.auth
com.app.notification
com.app.payment
```

## YAGNI EM API

```java
// ❌ endpoint com 30 query params "pra ser flexível"
GET /users?name=&email=&status=&role=&createdAfter=
          &createdBefore=&updatedAfter=&updatedBefore=
          &tenantId=&lang=&timezone=&sortBy=&sortDir=
          &page=&size=&includeDeleted=&includeInactive=

// Requisito atual: listar usuários ativos com paginação

// ✅ YAGNI
GET /users?page=0&size=20
// Adiciona filtro quando produto pede com caso de uso real
```

## Trade-offs
- YAGNI rigoroso = refatoração mais tarde
- Ignorar YAGNI = sistemas inchados

Heurística: Construa para os casos atuais, projete para mudar.