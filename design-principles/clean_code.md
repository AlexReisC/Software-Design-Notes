# Clean Code

## Nomes Significativos

### Definição

Nome deve revelar intenção. Lê -> entende.

**Ruim:**

```java
int d; // dias
List<int[]> list1;

public List<int[]> getThem() {
    List<int[]> l = new ArrayList<>();
    for (int[] x : list1)
        if (x[0] == 4) l.add(x);
    return l;
}
```

**Limpo:**

```java
int elapsedDays;
List<Cell> flaggedCells;

public List<Cell> getFlaggedCells() {
    List<Cell> flaggedCells = new ArrayList<>();
    for (Cell cell : gameBoard)
        if (cell.isFlagged()) flaggedCells.add(cell);
    return flaggedCells;
}
```

### REGRAS CORE

| Regra | ❌ | ✅ |
| ----- | -- | -- |
| Intenção clara | `d` | `elapsedDays` |
| Sem desinformação | `accountList` (não é List) | `accounts` |
| Distinção real | `data1, data2` | `userData, sessionData` |
| Pronunciável | `genymdhms` | `generatedTimestamp` |
| Buscável | `7` | `MAX_RETRIES = 7` |
| Sem prefixo húngaro | `strName, iCount` | `name, count` |
| Classe = substantivo | `ProcessData` | `DataProcessor` |
| Método = verbo | `name()` | `getName()` |

## FUNÇÕES PEQUENAS

### Definição

Função faz uma coisa. Faz bem. Só ela. Se precisa "e" pra descrever então quebra.

Ruim:

```java
public void processUserOrder(Order order) {
    // valida
    if (order.getItems().isEmpty()) throw new RuntimeException("Empty order");
    if (order.getUser() == null) throw new RuntimeException("No user");

    // calcula total
    double total = 0;
    for (Item item : order.getItems())
        total += item.getPrice() * item.getQuantity();
    order.setTotal(total);

    // aplica desconto
    if (order.getUser().isPremium()) order.setTotal(total * 0.9);

    // salva
    orderRepository.save(order);

    // envia email
    emailService.send(order.getUser().getEmail(), "Order confirmed: " + order.getId());
}
```

Problema → valida + calcula + desconta + persiste + notifica. **5 responsabilidades**.

Limpo:

```java
public void processUserOrder(Order order) {
    validateOrder(order);
    calculateTotal(order);
    applyDiscount(order);
    orderRepository.save(order);
    notifyUser(order);
}

private void validateOrder(Order order) {
    if (order.getItems().isEmpty()) throw new InvalidOrderException("Empty order");
    if (order.getUser() == null) throw new InvalidOrderException("No user");
}

private void calculateTotal(Order order) {
    double total = order.getItems().stream()
        .mapToDouble(item -> item.getPrice() * item.getQuantity())
        .sum();
    order.setTotal(total);
}

private void applyDiscount(Order order) {
    if (order.getUser().isPremium())
        order.setTotal(order.getTotal() * 0.9);
}

private void notifyUser(Order order) {
    emailService.send(order.getUser().getEmail(), "Order confirmed: " + order.getId());
}
```

### REGRAS CORE

| Regra | Detalhe |
| ----- | ------- |
| Tamanho | 5–20 linhas ideal. 20+ → sinal de alerta |
| Um nível de abstração | Não mistura `repository.save()` com `item.getPrice() * qty` |
| Sem side effects ocultos | Função `validateUser()` não deve alterar estado |
| Argumentos mínimos | 0–2 ideal. 3+ → considera objeto |
| Sem flag args | `process(true)` → `processPremium()` + `processStandard()` |

**NÍVEL DE ABSTRAÇÃO — conceito chave:**

```java
// ❌ mistura níveis
public void saveUser(User user) {
    String sql = "INSERT INTO users VALUES (?, ?)"; // baixo nível
    validateEmail(user.getEmail());                 // alto nível
    jdbcTemplate.update(sql, user.getName(), ...);  // baixo nível
}

// ✅ mesmo nível
public void saveUser(User user) {
    validateUser(user);        // alto
    persistUser(user);         // alto
    auditUserCreation(user);   // alto
}
```

## COMENTÁRIOS

### Definição

Comentário bom = explica por quê, não o quê. Código explica o quê. Se precisa comentar o quê → código ruim.
> "Every comment is a failure to express intent in code." — Robert C. Martin

#### COMENTÁRIOS RUINS

Ruído puro:

```java
// incrementa i
i++;

// construtor
public User() {}

// retorna nome
public String getName() { return name; }
```

Desatualizado = prio que nenhum:

```java
// valida email do usuário
public boolean validatePhone(String phone) {
    return phone.matches("\\d{10,11}");
}
```

Código comentado = NUNCA :

```java
// public void oldProcess() {
//     legacyService.run();
//     System.out.println("done");
// }
public void process() { ... }
```

#### COMENTÁRIOS BONS

Por quê (decisão não óbvia):

```java
// BCrypt limita input a 72 bytes. Senha maior → silenciosamente truncada.
// Hash feito antes do encode por segurança.
public String hashPassword(String password) {
    if (password.getBytes().length > 72)
        throw new InvalidPasswordException("Password exceeds 72 bytes");
    return encoder.encode(password);
}
```

Aviso de consequências:

```java
// ATENÇÃO: operação irreversível. Deleta dados do usuário em cascata.
public void deleteAccount(UUID userId) {
    userRepository.deleteWithCascade(userId);
}
```

TODO legítimo (rastreável):

```java
// TODO: substituir por cache Redis — ticket PROJ-482
public User findById(UUID id) {
    return userRepository.findById(id).orElseThrow();
}
```

Regex / algoritmo não trivial:

```java
// Valida CPF: 11 dígitos, rejeita sequências repetidas (111.111.111-11)
private static final String CPF_REGEX = "^(?!\\d)\\d{3}\\.\\d{3}\\.\\d{3}-\\d{2}$";
```

### REGRA PRÁTICA

| Situação | Ação |
| -------- | ---- |
| Explica o quê o código faz | Melhora o código, remove comentário |
| Explica decisão de negócio | Mantém |
| Código comentado | Deleta sempre |
| Desatualizado | Deleta ou atualiza |
| Aviso crítico | Mantém |
| Javadoc em API pública | Mantém |


Comentário bom = contexto que código não consegue expressar sozinho.

## FORMATAÇÃO & ESTRUTURA

### Definição

Formatação = comunicação. Time lê código mais que escreve. Consistência → velocidade de leitura. Discussão de estilo → automatiza com ferramenta, não com PR.

### FORMATAÇÃO VERTICAL

```java
// ✅ Ordem correta — lê de cima pra baixo naturalmente
public class OrderService {

    public void processOrder(Order order) {   // público, alto nível
        validateOrder(order);
        calculateTotal(order);
        save(order);
    }

    private void validateOrder(Order order) { // privado, detalhe
        if (order.isEmpty()) throw new InvalidOrderException();
    }

    private void calculateTotal(Order order) {
        order.setTotal(order.getItems().stream()
            .mapToDouble(Item::totalPrice).sum());
    }

    private void save(Order order) {
        repository.save(order);
    }
}
```

### DISTÂNCIA VERTICAL

```java
// ❌ Conceitos relacionados separados por linhas vazias desnecessárias
public void save(User user) {

    validate(user);


    repository.save(user);

    log.info("saved");
}

// ✅ Relacionados juntos. Separados por linha apenas quando muda conceito
public void save(User user) {
    validate(user);
    repository.save(user);
    log.info("User {} saved", user.getId());
}
```

### FORMATAÇÃO HORIZONTAL

```java
// ❌
if(user!=null&&user.isActive()&&user.hasRole("ADMIN")){doSomething(user,true,3);}

// ✅
if (user != null && user.isActive() && user.hasRole("ADMIN")) {
    doSomething(user, true, 3);
}
```

Limite de linha: 80–120 chars. Além → quebra. IDE horizontal scroll = ruído.

### ESTRUTURA DE CLASSE — ORDEM

```java
public class UserService {

    // 1. constantes
    private static final int MAX_LOGIN_ATTEMPTS = 5;

    // 2. campos estáticos
    private static final Logger log = LoggerFactory.getLogger(UserService.class);

    // 3. campos de instância
    private final UserRepository repository;
    private final PasswordEncoder encoder;

    // 4. construtor
    public UserService(UserRepository repository, PasswordEncoder encoder) {
        this.repository = repository;
        this.encoder = encoder;
    }

    // 5. métodos públicos
    public User create(UserRequest request) { ... }
    public void delete(UUID id) { ... }

    // 6. métodos privados (próximos de quem chama)
    private void validateEmail(String email) { ... }
    private void hashPassword(UserRequest request) { ... }
}
```

### REGRAS CORE

| Regra | Detalhe |
| ----- | ------- |
| Arquivo pequeno | 200–500 linhas ideal. 1000+ → alerta |
| Linha pequena | 80–120 chars |
| Conceitos relacionados | Próximos verticalmente |
| Conceitos distintos | Separados por linha vazia |
| Ordem de classe | Constantes → campos → construtor → público → privado |
| Indentação | Consistente. Nunca mistura tabs/spaces |

## TRATAMENTO DE ERRO

#### Definição
Erro = cidadão de primeira classe. Trata explícito, separado da lógica de negócio. Nunca silencia, nunca usa fluxo de controle.

**RUIM — exceção como fluxo de controle**

```java
public User findUser(UUID id) {
    try {
        return repository.findById(id);
    } catch (Exception e) {
        return null; // silencia erro
    }
}

// caller
User user = findUser(id);
if (user == null) { // NullPointerException esperando
    // ...
}
```

Problema → erro silenciado. Caller não sabe se null = "não existe" ou "explodiu".

**RUIM — checked exceptions vazando abstração**

```java
// Interface de alto nível expõe detalhe de impl
public interface UserRepository {
    User findById(UUID id) throws SQLException; // caller precisa saber de SQL?
}
```

**LIMPO — exceções específicas + hierarquia**

```java
// Hierarquia de domínio
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(UUID id) {
        super("User not found: " + id);
    }
}

public class UserServiceException extends RuntimeException {
    public UserServiceException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

```java
// Service limpo — lógica separada de erro
public User findUser(UUID id) {
    return repository.findById(id)
        .orElseThrow(() -> new UserNotFoundException(id));
}

public void createUser(UserRequest request) {
    validateRequest(request);
    try {
        repository.save(toEntity(request));
    } catch (DataIntegrityViolationException e) {
        throw new UserServiceException("Email already exists", e);
    }
}
```

**LIMPO — handler centralizado (Spring)**

```java
// Um lugar trata tudo. Service não sabe de HTTP.
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException ex) {
        return ResponseEntity.status(404)
            .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(UserServiceException.class)
    public ResponseEntity<ErrorResponse> handleServiceError(UserServiceException ex) {
        log.error("Service error", ex);
        return ResponseEntity.status(500)
            .body(new ErrorResponse("INTERNAL_ERROR", "Unexpected error"));
    }
}
```

### REGRAS CORE

| Regra | Detalhe |
| ----- | ------- |
| Nunca retorna null | Usa `Optional<T>` ou lança exceção |
| Nunca passa null | Valida entrada no boundary |
| Nunca silencia catch | Mínimo: `log.error()` + rethrow |
| Exceção específica | `UserNotFoundException` > `RuntimeException` |
| Unchecked > Checked | Checked vaza abstração. Unchecked = padrão |
| Erro ≠ fluxo de controle | `try/catch` não substitui `if` |
| Handler centralizado | Um lugar trata, não espalhado no código |

#### NUNCA SILENCIA
```java
// ❌ crime capital
try {
    riskyOperation();
} catch (Exception e) {
    // silêncio
}

// ✅ mínimo aceitável
try {
    riskyOperation();
} catch (Exception e) {
    log.error("riskyOperation failed for user {}", userId, e);
    throw new ServiceException("Operation failed", e);
}
```
