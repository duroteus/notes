### MVCC — o que realmente acontece dentro do PostgreSQL

Imagine uma tabela `users`:

| id  | name |
| --- | ---- |
| 1   | Ana  |

Agora duas transações acontecem ao mesmo tempo:

**Transação A**

```sql
BEGIN;
UPDATE users SET name = 'Ana Maria' WHERE id = 1;
-- ainda não deu commit
```

**Transação B (ao mesmo tempo)**

```sql
SELECT * FROM users WHERE id = 1;
```

Em muitos bancos tradicionais (modelo lock-based clássico), o `SELECT` ficaria bloqueando esperando o `UPDATE` terminar.

No PostgreSQL -> **isso não acontece**

O `SELECT` continua instantaneamente.

Por quê?

Porque o PostgreSQL **não altera a linha existente**.

Ele faz isso internamente:

| id  | name      | xmin | xmax |
| --- | --------- | ---- | ---- |
| 1   | Ana       | 100  | 200  |
| 1   | Ana Maria | 200  | NULL |

O banco literalmente mantém **duas versões físicas da mesma linha**.

- a versão antiga continua existindo
- a nova versão é a atualização

Cada transação recebe um **snapshot do banco no momento em que começa**.

Então

- quem começou antes do commit vê **Ana**
- quem começou depois vê **Ana Maria**

Isso é MVCC.

#### O problemaque ele resolve

Sem MVCC você teria:

**read locks + write locks**

Em sistemas web reais:

- APIs lendo usuários
- workers atualizando pedidos
- jobs recalculando saldo
- autenticação lendo sessão

Sem MVCC:
o banco vira um engarrafamento de locks.

Com MVCC:

- `SELECT` não bloqueia `UPDATE`
- `UPDATE` não bloqueia `SELECT`
- concorrência escala

Por isso PostgreSQL aguenta **muitas leituras simultâneas**.

#### O custo disso (muito importante)

Como o banco **não remove a versão antiga imediatamente**, ele acumula lixo.

Esse lixo são linhas que:

- não são mais visíveis para ninguém
- mas ainda estão ocupando espaço físico

### A consequência do MVCC — Dead tuples

Quando ocorre:

```sql
UPDATE users SET name = 'Ana Maria' WHERE id = 1;
```

O PostgreSQL:

**1.** cria nova versão da linha <br>
**2.** marca a antiga com `xmax = tx_id` <br>

Enquanto houver uma transação ativa que começou antes do commit, aquela versão antiga ainda pode ser visível.

Somente quando:

- todas as transações antigas terminam
- não há snapshot que precise daquela versão

ele vira um **dead tuple real**.

Ela não é removida automaticamente.
Ela só é marcada como reaproveitável.

Quem limpa:

- `VACUUM`
- `autovacuum`

Se VACUUM não rodar:

- tabela cresce indefinidamente
- índices também crescem
- performance cai progressivamente

Isso é um problema clássico em produção com:

- long transactions
- replicação atrasada
- carga alta de UPDATE/DELETE

### E agora? VACUUM!

#### O que o VACUUM realmente faz

Lembra das dead tuples?
Elas continuam dentro do arquivo físico da tabela (`heap file`).

O VACUUM:

**1.** varre página por página do heap <br>
**2.** verifica visibility (usando xmin/xmax) <br>
**3.** confirma que nenhuma transação precisa mais daquela versão <br>
**4.** marca o espaço como reutilizável (free space map) <br>

Importante:

> Ele não apaga a linha do arquivo.
> Ele só permite que uma próxima inserção utilize aquele espaço.

Por isso o arquivo não diminui.

#### Visibility Map (conceito importante)

Durante o VACUUM o PostgreSQL marca páginas como:

- **all-visible**
- **all-frozen**

Isso permite:

- index-only scan
- evitar novas varreduras caras

Ou seja: VACUUM também melhora leitura, não só escrita.

#### VACUUM FULL (por que é perigoso)

`VACUUM FULL` faz:

```
cria nova tabela
copia apenas linhas vivas
apaga antiga
renomeia
reindexa
```

Isso exige **ACCESS EXCLUSIVE LOCK**.

Na prática:

- API trava
- requisições acumulam
- pode derrubar o sistema

Só deve ser usado em manutenção planejada.

#### Autovacuum (o verdadeiro herói do PostgreSQL)

O PostgreSQL não depende do DBA manualmente.

O autovacuum:

- monitora o número de UPDATE/DELETE/INSERT
- quando passa do threshold -> roda VACUUM + ANALYZE

Se ele estiver:

- desativado
- com `cost_limit` muito baixo
- com tabela muito grande
  -> o banco lentamente degrada.

Este é o motivo n. 1 de "PostgreSQL ficou lento depois de meses em produção".

#### Transaction ID Wraparound (muito importante)

Cada transação recebe um `txid` de 32 bits.
Eles eventualmente **acabam**.

Se o banco não congelar linhas antigas via VACUUM:
o PostgreSQL entra em modo de proteção:

> ele começa a recusar escrita para evitar corrupção de dados

Ou seja:
**VACUUM não é otimização — é mecanismo de sobrevivência do banco**.

### Vamos falar de WAL (Write-Ahead Log)

#### O problema que o WAL resolve

Imagine:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

Entre o `COMMIT` e a gravação real no arquivo da tabela, ocorre:

**queda de energia**.

Sem WAL -> corrupção lógica;

- dinheiro saiu de uma conta
- não entrou na outra conta

O banco ficaria inconsistente.

#### O que o PostgreSQL faz

Antes de alterar a tabela física (`heap`), ele escreve no WAL algo como:

```
tx 500:
page 123: old tuple -> new tuple
page 456: old tuple -> new tuple
commit
```

Quando você da `COMMIT`, o PostgreSQL **não precisa salvar a tabela ainda**.

Ele só precisa garantir:

> o WAL foi gravado em disco com fsync.

Isso é rápido porque:

- é escrita sequencial
- não é acesso aleatório a páginas da tabela

#### Então por que gravar depois nas tabelas?

As tabelas são organizadas em páginas espalhadas no disco.
Gravar nelas diretamente a cada transação seria muito lento.

O PostgreSQL usa:

**WAL = garantia de segurança**
**data files = armazenamento final**

Se o servidor cair:
na próxima inicialização ele executa:

> crash recovery -> replay do WAL -> restaura o banco ao último commit confirmado.

#### Consequência importante

Quando você vê:

> "PostgreSQL demorando no commit"

geralmente não é CPU nem query.

É:
**latência de fsync no disco do WAL**

Por isso SSD/NVMe impacta muito PostgreSQL.

### E o que são os checkpoints?

Quando você executa:

```sql
UPDATE orders SET status = 'paid' WHERE id = 10;
```

O PostgreSQL:

**1.** escreve no WAL <br>
**2.** altera a página na **RAM** (shared_buffers) <br>

Ele **não grava imediatamente na tabela física**.

Isso é proposital — gravar em disco a cada query destruiria performance.
Essas páginas alteradas na memória são chamadas de _dirty pages_.

#### Então quando o banco grava de verdade no disco?

No **checkpoint**.

O processo de background (`checkpointer`) pega milhares de dirty pages e faz:

RAM -> data files (heap/index)

Depois disso:

- o banco sabe que aquele ponto está seguro
- WAL antigo pode ser reciclado

#### Por que isso gera lentidão?

Porque o checkpoint é:

- massivo
- I/O intensivo
- disputa disco com queries

Resultado típico:

- aumento de tempo de commit
- `SELECT` também piora (disputa cache do SO)
- fila de requisições na API

Em produção você observa picos de latência periódicos.
Isso geralmente é checkpoint mal ajustado.

#### Relação direta com WAL

O WAL cresce continuamente

Checkpoint é o evento que diz:

> "tudo até este LSN já está persistido nas tabelas".

Então o PostgreSQL pode reaproveitar os segmentos WAL antigos.

Sem checkpoint:

- WAL cresce indefinidamente
- recovery ficaria gigantesco

### O que significa o acrônimo ACID?

#### A — Atomicidade

Exemplo clássico:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

Se cair entre as duas instruções

- WAL permite rollback automático
- ou replay consistente no recovery

Nunca fica "meio aplicada"

#### C — Consistência

Significa:

- integridade referencial
- regras de domínio
- constraints

Se você tentar:

```sql
INSERT INTO orders (user_id) VALUES (9999);
```

E o user **não existir** -> <span style="color:red">erro</span>.

A transação falha.

Consistência depende:

- do modelo
- das constraints declaradas
- da lógica da aplicação

#### I — Isolamento

Problemas clássicos:

- dirty read
- non-repeatable read
- phantom read

No PostgreSQL:

- não existe dirty read (nem em READ COMMITED)
- MVCC fornece snapshot isolation

O MVCC permite:

- `SELECT` não bloquear `UPDATE`
- cada transação ver seu próprio snapshot

#### D — Durabilidade

Após:

```sql
COMMIT;
```

O PostgreSQL:

- força WAL no disco
- só então retorna sucesso

Se houver crash:

- recovery executa WAL
- banco volta ao último commit confirmado

### E quais são os niveis de isolamento no PostgreSQL?

#### Os níveis no PostgreSQL

O padrão SQL define: <br>
**1.** READ UNCOMMITTED <br>
**2.** READ COMMITTED <br>
**3.** REPEATABLE READ <br>
**4.** SERIALIZABLE <br>

O PostgreSQL na prática oferece **3 comportamentos reais:**

**READ COMMITED (default)**
Cada query pega um snapshot novo.

Exemplo:

Transação A:

```sql
BEGIN;
SELECT count(*) FROM orders; -- 10
```

Transação B:

```sql
INSERT INTO orders VALUES (...);
COMMIT;
```

Transação A:

```sql
SELECT count(*) FROM orders; -- 11
```

A mesma transação viu dados diferentes.

**REPEATABLE READ**
O snapshot é fixo desde o `BEGIN`.

A transação sempre vê a mesma visão do banco, mesmo que outros deem commit.

Implementado com MVCC -> não usa lock pesado.

**SERIALIZIBLE**
O PostgreSQL usa **SSI (Serializible Snapshot Isolation)**.

Ele não bloqueia tudo; ele detecta conflitos perigosos e **aborta uma das transações** com:

> `could not serialize access due to concurrent update`

Ou seja:
o banco prefere abortar do que permitir inconsistência lógica.

**Phantom Read (muito perguntando)**
Phantom read não é "ler valor diferente".

É:

> novas linhas aparecem ao repetir uma query

Exemplo:

```sql
SELECT * FROM orders WHERE price > 100;
```

Outra transação insere um novo pedido com `price = 150`.

Você executa novamente -> aparece uma linha nova -> phantom.

Isso acontece porque MVCC controla versões de linhas, **não o conjunto lógico do resultado**.

Somente **SERIALIZIBLE** impede.

**Impacto prático em backend**
Problemas clássicos:

- dupla cobrança
- estoque negativo
- criação duplicada de registro

APIs financeiras geralmente usam:
`SERIALIZIBLE` ou locking explicito.

### O que é deadlock e o que fazer para previnir?

#### Exemplo real

Tabela `accounts`.

Transação 1:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- segura lock na linha 1
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- espera linha 2
```

Transação 2 (simultâneamente):

```sql
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;
-- segura lock na linha 2
UPDATE accounts SET balance = balance + 50 WHERE id = 1;
-- espera linha 1
```

Agora temos:

- T1 espera T2
- T2 espera T1

Nenhuma pode avançar.

Isso é um ciclo de espera -> deadlock

#### O que o PostgreSQL faz?

Ele não deixa travado para sempre.

Existe um processo interno que:

- analisa o grafo de dependências
- identifica o ciclo
- escolhe uma transação como vítima
- faz rollback dela

A outra continua normalmente.

Importante:

> O banco não "resolve" o conflito. Ele aborta alguém.

A aplicação precisa estar preparada para retry.

#### Impacto em produção

Deadlock aparece mais em:

- sistemas financeiros
- controle de estoque
- filas em banco
- APIs concorrentes em Node.js

Se você não tratar retry

- usuário recebe erro 500
- operação falha mesmo sendo válida

### E quais são os tipos de locks?

:one: **Row-Level Locks (mais comuns)**

Quando você faz:

```sql
UPDATE orders SET status = 'paid' WHERE id = 10;
```

O PostgreSQL:

- cria nova versão (MVCC)
- coloca **lock na linha antiga**

Outras transações que tentarem atualizar a mesma linha:

- ficam esperando
- ou geram conflito

Tipos importantes:

- `FOR UPDATE`
- `FOR NO KEY UPDATE`
- `FOR SHARE`
- `FOR KEY SHARE`

Uso típico:

- controle de estoque
- evitar dupla cobrança
- processamento de fila

:two: **Table-Level Locks**

Sempre que você executa:

```sql
ALTER TABLE users ADD COLUMN age INT;
```

O banco aplica `ACCESS EXCLUSIVE LOCK`.

Isso:

- bloqueia `SELECT`
- bloqueia `INSERT`
- bloqueia `UPDATE`

Impacto:

- downtime temporário

Por isso migrations mal planejadas derrubam produção.

:three: **Advisory Locks**

Exemplo:

```sql
SELECT pg_advisory_lock(12345);
```

Eles:

- não bloqueiam linhas automaticamente
- são apenas um mecanismo de sincronização

Uso real:

- evitar que dois workers processem o mesmo job
- lock distribuido simples
- controle de execução única

Muito usado com Node.js + múltiplas instâncias.

#### Impacto em backend (caso real)

Em uma API Node.js com pool de conexões:

- muitas requisições concorrentes
- cada uma abre transação
- atualiza mesma tabela

Se a ordem de acesso for inconsistente -> deadlock.
Se transações forem longas -> lock rentention alto.
Se houver DDL em horário errado -> freeze da aplicação.

### E como o PostgreSQL busca os dados nas tabelas?

O PostgreSQL tem um componente crítico:

> **query planner (optimizer)**

Ele não pergunta: "_tem índice?_"
Ele pergunta: "_qual plano custa menos I/O?_"

#### Sequential Scan

```sql
SELECT * FROM users;
```

O banco faz:

page 1 -> page 2 -> page 3 -> ... -> fim

Isso é extremamente eficiente para disco/SSD porque:

- leitura sequencial
- prefetch do sistema operacional
- bom uso de cache

#### Index Scan

Se existir

```sql
CREATE INDEX idx_users_email ON users(email);
```

E você fizer:

```sql
SELECT * FROM users WHERE email = 'ana@email.com';
```

O banco:

**1.** consulta a B-tree <br>
**2.** encontra ponteiro da linha <br>
**3.** vai até o heap buscar a tupla <br>

Problema:
isso gera **acesso aleatório ao disco.**

#### Por que às vezes o índice piora

Imagine:

```sql
SELECT * FROM orders WHERE status = 'paid';
```

Se 70% da tabela tiver `paid`:

Usar índice faria:

- milhares de seeks aleatórios
- milhares de leituras do heap

Seq Scan:

- uma única varredura contínua

Logo, o planner prefere Seq Scan.

#### O fator decisivo: seletividade

Alta seletividade -> índice
Baixa seletividade -> seq scan

Exemplos ruins para índice:

- `status`
- `ativo = true`
- `sexo`
- colunas boleanas

Exemplos bons:

- email
- CPF
- id
- uuid

#### Estastísticas

O PostgreSQL mantém estastísticas (`ANALYZE`) sobre distribuição de valores.

Se estivem desatualizadas:
o planner escolhe plano errado.

Sintomas clássico:

> query ficou lenta sem mudar código.
