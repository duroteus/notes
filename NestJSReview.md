# NestJS

#### 1) O que é o NestJS e qual problema ele tenta resolver dentro do ecossistema Node.js?

**O NestJS é um framework backend para Node.js que resolve a falta de arquitetura e padronização presente em frameworks minimalistas como Express.** Ele fornece um container de injeção de dependência, modularização e ciclo de vida de aplicação, inspirado principalmente no Angular e em frameworks como Spring Boot. Ele roda sobre adapters HTTP como Express ou Fastify, atuando como uma camada de abstração. O uso de decorators permite registrar metadados e construir automaticamente rotas, providers e dependências, tornando o código mais testável, previsível e escalável.

---

#### 2) O que é um Provider no NestJS e como o mecanismo de Dependency Injection realmente funciona internamente?

Um provider no NestJS é **qualquer classe registrada no container de injeção de dependência (IoC container)** do módulo e gerenciada pelo ciclo de vida da aplicação. **Normalmente, services, repositories e factories são providers**. Ao usar **<span style="color: green">@Injectable()</span>**, a classe passa a poder ser resolvida pelo container.

Quando a aplicação inicia (**<span style="color: green">NestFactory.create</span>**), o Nest cria um *Application Context* e faz um scan dos decoratores (**<span style="color: green">@Module, @Injectable, @Controller</span>**) usando **<span style="color: green">reflect-metadata</span>**. A partir disso ele constrói um grafo de módulos e dependências.

Cada provider é registrado no container com um token (por padrão, a própria classe). O Nest ainda não instancia os objetos nesse momento.

Quando algum controller ou provider precisa de uma dependência no **<span style="color: green">constructor</span>**, o Nest:
  - Lê os tipos dos parâmetros do constructor via metadata (**<span style="color: green">design:paramtypes</span>**)

  - Identifica quais providers são necessários.

  - Resolve recursivamente o grafo de dependências.

  - Instancia primeiro as dependências internas.

  - Armazena as instâncias do container (cache singleton por padrão).

  - Injeta a instância já criada no objeto que está sendo contruído.

Assim, o desenvolvedor não cria objetos manualmente (**<span style="color: green">new</span>**). O container cria e compartilha instâncias entre toda a aplicação, garantindo desacoplamento e testabilidade.

---

#### 3) O NestJS possui três escopos:
 - `DEFAULT`
 - `REQUEST`
 - `TRANSIENT`
   
Explique:
  **1.** O que cada um significa?
  **2.** O que acontece em uma API com 100 requisições simultâneas?
  **3.** Qual deles pode derrubar a performance de uma aplicação se usado incorretamente e por quê?

#### <span style="color: green">DEFAULT</span> (Singleton - padrão)

 - 1 instância para toda a aplicação
 - 100 requisições -> continuam usando a mesma insância.
 - Melhor perofrmance e menor uso de memória -> Deve ser o escopo da maioria dos services (DB, cache, APIs externas, repos).

#### <span style="color:green">REQUEST</span>

 - 1 instância nova por requisição HTTP
 - 100 requisições -> 100 instâncias (e também das dependências internas)
 - Aumenta a latência, memória e pressão no GC -> Use só quando precisar de estado por request (ex: usuário autenticado, tenant, correlationId).

#### <span style="color:green">TRANSIENT</span>
 - Nova instância toda vez que for injetado, mesmo dentro da mesma request.
 - Pode criar muitas instâncias rapidamente. -> Uso raro; geralmente apenas helpers sem estado.

**Qual mais prejudica performance?** **<span style="color: green">REQUEST</span>**, pois recria toda a arvore de dependências por requisição.

---

#### 4) Explique a responsabilidade de:
 - **Middleware**
 - **Guards**
 - **Pipes**
 - **Interceptors**

**e um caso típico de uso para cada**.

**Middleware:** executa antes do Nest e atua no nível do HTTP adapter (Express/Fastify). Não conhece o handler. É usado para tarefas genéricas como logging simples, manipulação de headers, cookies e CORS.

**Guards:** decidem se a rota pode ser executada. Recebem o **<span style="color:green">ExecutionContext</span>** e normalmente implementam autenticação e autorização, como validação de JWT ou verificação de roles.

**Pipes:** validam e transforam os parâmetros do método do controller antes da execução. São usados para validação de DTOs (**<span style="color:green">ValidationPipe</span>**) ou conversão dos tipos (**<span style="color:green">ParseIntPipe</span>**)

**Interceptors:** envolvem a execução do handler (antes e depois). Permitem lógica transversal como logging e tempo de execução, cache, tratamento de timeout/retry e padronização da resposta da API.

---

#### 5) Qual a diferença prática entre Exception Filter e Interceptor, e quando você usaria cada um?

**Interceptor** é usado para lógica transversal ao redor da execução do handler (antes e depois). Ele recebe o fluxo da resposta e pode transformá-la, medir tempo, aplicar cache, retry ou timeout. Atua no caminho normal de sucesso da requisição.

**Exception Filter** é usado exclusivamente para tratar erros lançados durante a execução. Ele captura exceções (**<span style="color:green">throw</span>**) e define qual resposta HTTP será enviada ao cliente (status code, payload, formato do erro)

Quando usar:
 - **Use interceptor** quando quiser alterar ou observar a execução da rota sem erro (ex: logging, mapear `{ data: ... }`, cache).

 - **Use Exception Filter** quando precisar padronizar ou customizar respostas de erro (ex: transformar erro de banco em **<span style="color:green">409</span>**, remover stacktrace, formato único de erro da API).

---

#### 6) Explique:
 - `useClass`
 - `useValue`
 - `useFactory`
   
**e um caso típico de uso para cada.**

No Nest, o `providers: []` não serve só para classes. Ele registra como o container deve resolver uma dependência. Essas três estratégias existem para controlar quem cria o objeto.

## <span style="color:green">useClass</span>

O container instancia automaticamente uma classe

```javascript
{
  provide: PaymentService,
  useClass: StripePaymentService,
}
```

Quando alguém pedir **<span style="color:green">PaymentService</span>**, o Nest cria **<span style="color:green">StripePaymentService</span>**

Uso típico:
Trocar implementação sem mudar quem consome (polimorfismo / strategy).

Exemplo real:
 - produção -> Stripe
 - teste -> MockPaymentService

## <span style="color:green">useValue</span>

Você fornece uma instância já pronta

```javascript
{
  provide: 'CONFIG'
  useValue: { 
    apiKey: 'abc',
    timeout: 5000,
  },
}
```

O Nest não instancia nada — ele só entrega esse objeto.
Uso típico:
 - config
 - constantes
 - mocks em testes
 - bibliotecas que não são classes do Nest.

## <span style="color:green">useFactory</span>

Você fornece uma função que cria o objeto.

```javascript
{
  provide: DatabaseClient,
  useFactory: async (config: ConfigService) => {
    return new PrismaClient({ url: config.dbUrl });
  },
  inject: [ConfigService],
}
```
O container executa a factory e usa o retorno como provider
Uso típico (muito importante):
 - inicialização assíncrona
 - conexão com banco
 - SDKs externos
 - objetos dependentes de configuração

Como diferenciar rapidamente:
 - useClass -> Nest instancia uma classe
 - useValue -> Você entrega um objeto pronto
 - useFactory -> você controla a criação

---

#### 7) O que exatamente é um Module no Nest e qual problema arquitetural ele resolve dentro de uma aplicação grande?

Um Module no NestJS é uma unidade de organização e encapsulamento do container de injeção de dependência.
Ele define um *escopo de providers* e controla quais dependências ficam privadas e quais podem ser compartilhadas com outros módulos.

Cada módulo possui seu próprio contexto de DI.
Providers declarados nele só podem ser usados internamente, a menos que sejam explicitamente exportados.

```javascript
@Module({
  providers: [UsersService],
  controllers: [UsersController],
  exports: [UsersService]
})
export class UserModule {}
```

Aqui, apenas **<span style="color:green">UsersService</span>** pode ser consumido por outros módulos que importarem **<span style="color:green">UsersModule</span>**

**Qual problema ele resolve?**

Sem módulos, toda a aplicação viraria um container global:
 - qualquer classe acessa qualquer dependência
 - alto acoplamento
 - impossível de escalar equipe
 - testes dificeis

O Module cria fronteiras arquiteturais (bounded contexts) dentro do backend.

Ele força:
 - separação de domínio
 - baixo acoplamento
 - reuso controlado
 - organização por feature (feature-based architecture)

> Module não é apenas uma pasta organizacional; ele define o escopo do container de dependências e cria fronteiras de encapsulamento entre partes da aplicação.

---

#### 8) Por que às vezes precisamos usar forwardRef() ao importar módulos ou injetar services?

**O que está acontecendo quando isso ocorre?**

**<span style="color:green">forwardRef()</span>** é usado para resolver dependências circulares no container de injeção de dependência.

Exemplo:


**UserService -> precisa de OrdersService**
**OrdersService -> precisa de UsersService**

```javascript
@Injectable()
export class UsersService {
  constructor(private ordersService: OrdersService) {}
}

@Injectable()
export class OrdersService {
  constructor(private usersService: UsersService) {}
}
```

Quando o Nest sobe a aplicação, ele tenta montar o grafo de dependências.

O que acontece internamente:

-> **criar UserSerice**
-> **precisa de OrdersSerice**
-> **criar OrdersService**
-> **precisa de UsersService**
-> **<span style="color:red">ainda não existe</span>**

 O container não consegue completar a resolução topológica do grafo (não é loop de execução, é impossibilidade de construção do objeto).

 Então o erro clássico:
 `Nest can´t resolve dependencies of the UsersService`

 **O que o <span style="color:green">forwardRef()</span> faz de verdade**

 Ele cria uma referência atrasada (deferred reference) para o token de DI.

 ```javascript
 constructor(
   @Inject(forwardRef(() => OrdersService))
   private ordersService: OrdersService,
 ) {}
``` 
O container passa a:
 - registrar primeiros os dois providers
 - criar as instancias parcialmente
 - completar a injeção depois que ambos existirem

Ou seja, ele quebra o ciclo no momento da resolução do container, não no runtime da aplicação.

**Quando usar**:
**<span style="color:green">forwardRef</span>** resolve dependência circular, mas normalmente indica um problema de arquitetura. O ideal é extrair a lógica compartilhada para um terceiro service ou módulo para remover o acoplamento bidirecional.

**Exemplo correto:**
UsersService <-> OrdersService :exclamation::exclamation: 
            ↓
UserOrdersDomainService :white_check_mark:

---

#### 9) No NestJS, qual a diferença entre DTO e Entity e por que misturar os dois é considerado má prática?

**DTO** (Data Transfer Object) representa o formato dos dados que entram ou saem da API. Ele pertence à camada de transporte (HTTP) e é usado principalmente para validação e tipagem da interface pública da aplicação, normalmente junto ao **<span style="color: green">ValidationPipe</span>**

**Entity** representa o modelo do domínio persistido. Ele pertence à camada de persistência e define como os dados existem no banco (ORM/ODM), incluindo relacionamentos, colunas e regras de armazenamento.

**Por que não misturar?**
DTO define contrato da API.
Entity define modelo de dados interno.

Se você reutiliza Entity como DTO:
 - Você expõe detalhes internos do banco.
 - quebra versionamento da API.
 - cria risco de segurança
 - acopla a camada HTTP ao ORM.

**Exemplo clássico**
A entidade possui:
 - `passwordHash`
 - `isAdmin`
 - `deletedAt`

Você não quer que isso saia automaticamente numa resposta.

**Outro problema sério**
Uma mudança no banco:
`username -> login`
passaria a quebrar clientes da API

> DTO é contrato externo da aplicação.
> Entity é modelo interno de persistência.
> Separar os dois mantém desacoplamento entre a camada HTTP e a camada de dados.

---

#### 10) No NestJS, por que não é recomendado acessar diretamente <span style="color:green">req</span> e <span style="color:green">res</span> (Express/Fastify) dentro do controller sempre que possível?

**O que você perde ao fazer isso?**
No Nest, o controller deveria trabalhar em alto nível de abstração. Quando você usa diretamente **<span style="color:green">@Req</span>** ou **<span style="color:green">@Res</span>**, você sai do fluxo gerenciado pelo framework e passa a depender do HTTP adapter (Express/Fastify).

**Exemplo:**
```javascript
@Get()
get(@Res() res) {
  res.json({ ok: true })
}
```

Isso funciona — mas você acabou de *bypassar* o Nest.

**O que você perde?**
 - Interceptors deixam de funcionar corretamente. **(ex: response mapping, cache interceptor, timeout, logging de resposta)**.
 - Pipes e serialização automática **(<span style="color:green">ClassSerializerInterceptor</span>, <span style="color:green">ValidationPipe</span> de saída)**.
 - Portabilidade **(Seu código passa a ser específico de Express. Trocar para Fastify quebra o controller)**.
 - Testabilidade **(Fica difícil fazer testes unitários sem mockar <span style="color:green">res</span>)**.
 - Consistência de resposta **(O Nest não consegue mais padronizar o retorno HTTP)**.


 **Forma correta:**
 O controller deve retornar dados, não responder manualmente:
 
 ```javascript
 @Get()
 get() {
   return { ok: true };
 }
 ```

 **O Nest decide:**
 
  - status code
  - serialização
  - headers
  - interceptors

**Quando é aceitável usar o <span style="color:green">req</span>/<span style="color:green">res</span>?**

Casos específicos:
 - download/stream de arquivo
 - SSE (server-sent events)
 - manipulação manual de cookies/headers muito específica

> Ao usar <span style="color:green">req</span>/<span style="color:green">res</span> diretamente eu *bypasso* o ciclo de vida do Nest e perco interceptors, serialização e independência do adapter HTTP, além de dificultar testes.

---

#### 11) Qual a diferença entre Middleware e Interceptor se ambos conseguem executar código antes da rota?

**Por que o Nest criou dois mecanismos?**

Embora ambos executem antes da rota, eles atuam em niveis diferentes do pipeline.

**Middleware** roda no nível do HTTP adapter (Express/Fastify). Ele não conhece qual controller ou método será chamado e só tem acesso a <span style="color:green">req</span>, <span style="color:green">res</span> e <span style="color:green">next</span>. É usado para tarefas genéricas de protocolo HTTP, como logging básico, manipulação de headers, cookies, CORS e rate limit simples.

**Interceptor** roda dentro do contexto do Nest e conhece o <span style="color:green">ExecutionContext</span> e o handler. Ele envolve a execução do método do controller e também atua após a resposta.
É usado para lógica transversal da aplicação, como logging de tempo de execução, cache, timeout/retry e transformação/padronização da resposta.

**Por que existem os dois?**
O middleware resolve preocupações de infraestrutura HTTP, enquanto o interceptor resolve preocupações da aplicação. Separar os dois permite manter o controller desacoplado tanto do protocolo quanto da lógica transversal.

---

#### 12) No NestJS existe o conceito de Lifecycle Hooks. O que é o <span style="color:green">OnModuleInit</span> e em que tipo de situação real você usaria isso?

**<span style="color:green">OnModuleInit</span>** é um lifecycle hook executado pelo Nest logo após o container de dependências terminar de instanciar os providers de um módulo e resolver todas as injeções.

**Ou seja:**
-> providers criados
-> dependencias resolvidas
-> **<span style="color:green">OnModuleInit</span>** é chamado

Ele garante que todas as dependências do service já existem e podem ser usadas com segurança

```javascript
@Injectable()
export class CacheWarmupService implements OnModuleInit {
  async onModuleInit() {
    await this.loadInitialCache();
  }
}
```

**Para que ele realmente serve?**
Não é para criar dependencias — o DI já faz isso.
É para executar rotinas de inicialização que dependem dessas dependências.

Usos corretos:
 - aquecer cache (carregar dados frequentes)
 - validar configuração externa
 - registrar handlers em filas (RabbitMQ/Kafka)
 - sincronizar estado inicial em outro serviço
 - pré-carregar permissões/roles
 - iniciar scheduler interno

Exemplo real:
```javascript
@Injectable()
export class PermissionsService implements OnModuleInit {
  constructor(private repo: RolesRepository() {}

  async onModuleInit() {
    this.roles = await this.repo.loadAll();
  }
}
```

O erro comum
Muita gente tenta abrir conexão manual com o banco dentro do **<span style="color:green">OnModuleInit</span>**.

Isso geralmente é errado porque:
 - você cria dependência fora do container
 - perde controle de lifecycle
 - dificulta shutodnw gracioso

O correto seria:
`useFactory async -> cria conexão -> provider singleton`

> **<span style="color:green">OnModuleInit</span>** executa após o container resolver as dependências e serve para rotinas de inicialização que precisam de outros providers já disponíveis, não para criar recursos de infraestrutura.

---

#### 13) Explique como funciona a autenticação com JWT usando Passport no NestJS e qual o papel da Strategy e do Guard

No NestJS a autenticação não é feita diretamente no controller.
Ela é delegada ao `AuthGuard` que utiliza uma *Passport Strategy*.

A `JwtStrategy` é responsável por:

**1.** extrair o token da requisição (Authorization Bearer)
**2.** validar a assinatura
**3.** decodificar o payload
**4.** retornar o usuário no método `validate()`

O retorno do `validate()` não vai para a resposta HTTP
Ele é anexado em `req.user`.

O `AuthGuard('jwt')` intercepta a requisição antes do controller.
Se a strategy validar o token -> a rota executa
Se não validar -> 401

Fluxo real:
```
request -> AuthGuard -> JwtStrategy -> validate() -> req.user -> controller
```

O controller nunca valida token manualmente.

A responsabilidade fica separada:
- **Strategy** -> autentica (verifica identidade)
- **Guard** -> autoriza acesso à rota

Isso permite trocar o método de autenticação sem alterar controllers.

>No NestJS a autenticação com JWT é implementada usando Passport Strategies.
O `AuthGuard('jwt')` intercepta a requisição e delega a validação para a `JwtStrategy`, que extrai o token do header Authorization, valida a assinatura e executa o método `validate()`. O retorno desse método é anexado ao `request.user`.
>
>Se a validação for bem-sucedida o guard permite o acesso ao controller, caso contrário retorna 401. A Strategy autentica o usuário e o Guard protege a rota, mantendo os controllers desacoplados da lógica de autenticação.

---

#### 14) JWT é stateless. Então como é possível invalidar a sessão de um usuário antes do token expirar?

Esse é um dos maiores problemas do JWT

O servidor não guarda sessão.
Logo, depois de emitido, o token continua válido até o `exp`.

Se o usuário:
- trocar senha
- fizer logout
- for banido

o token ainda funcionaria.

Por isso precisamos de **estratégias de revogação**

As principais:

**1) Blacklist (revocation list)**
Guardar o `jti` (token id) no Redis:

```
logout -> salvar jti no redis
request -> verificar se jti está revogado
```

Prós:
- invalida imediatamente

Contras:
- perde parte do benefício stateless

**2) Short-lived Access Token + Refresh Token (padrão atual)**
- Access Token -> 5~15 min
- Refresh Token -> dias/semanas

Logout:
-> apaga refresh token do banco

O usuário não consegue gerar novos access tokens.

Essa é a estratégia mais usada em produção

**3) Rotação de Refresh Token**

Cada refresh gera outro refresh e invalída o anterior.

Protege contra:
- token roubado
- replay attack

**4) Versionamento do token (tokenVersion)**
Usuário tem no banco:

```
tokenVersion: 3
```

Token contém:

```
{ sub: userId, ver: 3 }
```

Ao trocar senha:

```
tokenVersion++
```

Tokens antigos para de funcionar.

> Como o JWT é stateless ele não pode ser invalidado diretamente pelo servidor após emitido. Para revogar sessões usamos estratégias complementares: blacklist de tokens (ex: armazenar `jti` em Redis), access tokens de curta duração combinados com refresh tokens persistos, rotação de refresh tokens ou versionametno de token no banco.
>
> O método mais comum em produção é access token curto + refresh token armazenado no banco, pois permite logout e revogação sem consultar o banco em toda requisição.