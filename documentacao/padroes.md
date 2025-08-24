# Princípios e Padrões de Projeto — Adote Fácil

Este documento apresenta a análise dos **princípios SOLID** e os **padrões de projeto** aplicados (ou que poderiam ser aplicados) no sistema **Adote Fácil**.

---

## 1. Princípios SOLID

### **S — Single Responsibility Principle (SRP)**

**Aplicado.**  
Cada camada tem responsabilidade única:

- Controllers → lidar com HTTP e repassar a lógica.
- Services → regras de negócio.
- Repositories → acesso a dados via Prisma.
- Providers → infraestrutura (ex.: autenticação, encriptação).

Exemplo:  
`controllers/user/create-user.ts` apenas extrai dados da requisição e chama o `CreateUserService`.

---

### **O — Open/Closed Principle (OCP)**

**Parcialmente aplicado.**  
A lógica pode ser estendida criando novos Services ou Repositories, mas há forte acoplamento a instâncias concretas (`userRepositoryInstance`).

**Sugestão:** usar interfaces (`IUserRepository`) para permitir a troca de implementações sem alterar código cliente.

---

### **L — Liskov Substitution Principle (LSP)**

**Neutro.**  
Não há hierarquias complexas de herança.

Se interfaces forem introduzidas, é importante manter compatibilidade entre implementações.

---

### **I — Interface Segregation Principle (ISP)**

**Parcialmente aplicado.**  
Os DTOs (`*.dto.d.ts`) já ajudam a manter contratos pequenos.

Sugere-se dividir interfaces de repositórios para evitar métodos “gordos” não utilizados.

---

### **D — Dependency Inversion Principle (DIP)**

**Parcial.**  
Há injeção de dependência via construtor (ex.: Services recebem UserRepository e Encrypter), mas ainda baseada em classes concretas.

**Sugestão:** introduzir abstrações (interfaces) e um container de injeção (Awilix, Tsyringe) para maior flexibilidade.

---

## 2. Padrões Identificados

### **1) Repository Pattern — Aplicado**

Os repositórios isolam o Prisma da lógica de negócio.

Exemplo simplificado:

```ts
// repositories/user.ts
export class UserRepository {
  constructor(private readonly repository: PrismaClient) {}

  async create(params: CreateUserRepositoryDTO.Params) {
    return this.repository.user.create({ data: params });
  }
}
```

**Benefício:** separa acesso a dados da regra de negócio, facilita testes e manutenção.

---

### **2) Middleware (Chain of Responsibility) — Aplicado**

Usado para autenticação, upload e tratamento de erros no Express.

Exemplo:

```ts
// middlewares/user-auth.ts
class UserAuthMiddleware {
  constructor(private readonly authenticator: Authenticator) {}

  authenticate(req: Request, res: Response, next: NextFunction) {
    const authHeader = req.headers.authorization;
    const [, token] = authHeader.split(" ");
    const decoded = this.authenticator.validateToken(token);
    if (!decoded) return res.status(401).json({ message: "Token inválido" });
    req.user = decoded;
    return next();
  }
}
```

**Benefício:** adiciona funcionalidades transversais sem poluir os controllers.

---

### **3) Singleton — Aplicado**

Instâncias únicas de serviços utilitários são exportadas e reutilizadas.

Exemplo:

```ts
// providers/authenticator.ts
export const authenticatorInstance = new Authenticator();
```

**Benefício:** evita recriação desnecessária e mantém consistência no uso.

---

## 3. Padrões Sugeridos

### **A) Strategy — Sugerido**

Aplicável para regras variáveis de filtragem/listagem de animais.

Exemplo:

```ts
export interface AnimalFilterStrategy {
  apply(params: any): Promise<Animal[]>;
}

export class AvailableAnimalsStrategy implements AnimalFilterStrategy {
  apply(params) {
    /* busca animais disponíveis */
  }
}

export class ByTypeAnimalsStrategy implements AnimalFilterStrategy {
  apply(params) {
    /* busca por tipo */
  }
}
```

**Benefício:** facilita adicionar novos filtros sem alterar a lógica central.

---

### **B) Factory (ou Abstract Factory) — Sugerido**

Pode ser usada para centralizar a criação de Services com suas dependências.

Exemplo:

```ts
export class CreateUserFactory {
  static make() {
    return new CreateUserService(encrypterInstance, userRepositoryInstance);
  }
}
```

**Benefício:** simplifica a criação de objetos complexos e reduz acoplamento.

---

## 4. Conclusão

O projeto aplica bem **SRP**, **Repository Pattern**, **Middleware** e **Singleton**.

Há espaço para evoluir em **DIP/ISP**, introduzindo interfaces e containers de injeção.

Padrões como **Strategy** e **Factory** podem aumentar a extensibilidade e facilitar manutenção futura.
