- [MVCC — o que realmente acontece dentro do PostgreSQL](#mvcc-o-que-realmente-acontece-dentro-do-postgresql)
- [A consequência do MVCC — Dead tuples](#a-consequencia-do-mvcc-dead-tuples)
- [E agora? VACUUM!](#e-agora-vacuum)
- [A consequência do MVCC — Dead tuples](#a-consequencia-do-mvcc-dead-tuples)
- [E o que são os checkpoints?](#e-o-que-sao-os-checkpoints)
- [O que significa o acrônimo ACID?](#o-que-significa-o-acronimo-acid)
- [E quais são os niveis de isolamento no PostgreSQL?](#e-quais-sao-os-niveis-de-isolamento-no-postgresql)
- [O que é deadlock e o que fazer para previnir?](#o-que-e-deadlock-e-o-que-fazer-para-previnir)
- [E quais são os tipos de locks?](#e-quais-sao-os-tipos-de-locks)
- [E como o PostgreSQL busca os dados nas tabelas?](#e-como-o-postgresql-busca-os-dados-nas-tabelas)
- [Como o PostgreSQL decide executar uma query](#como-o-postgresql-decide-executar-uma-query)
- [Índices do PostgreSQL](#indices-do-postgresql)
- [Paginação: OFFSET x Keyset](#paginacao-offset-x-keyset)
- [Pool de conexões](#pool-de-conexoes)
- [O problema de transações longas](#o-problema-de-transacoes-longas)
- [Replicas](#replicas)
- [Failover](#failover)
- [Migrations](#migrations)
- [Prepared Statements](#prepared-statements)
- [O problema de N+1 queries](#o-problema-de-n1-queries)
- [Idempotência no PostgreSQL](#idempotencia-no-postgresql)
- [Fila usando PostgreSQL](#fila-usando-postgresql)
- [O que é pg_stat_activity e pg_stat_statements e qual sua importância?](#o-que-e-pg_stat_activity-e-pg_stat_statements-e-qual-sua-importancia)
- [O que é PITR (Point-in-Time Recovery)?](#o-que-e-pitr-point-in-time-recovery)
- [Quando usar replicação física e quando usar replicação lógica](#quando-usar-replicacao-fisica-e-quando-usar-replicacao-logica)
- [Contenção de locks](#contencao-de-locks)

---

<a id="mvcc-o-que-realmente-acontece-dentro-do-postgresql"></a>

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

No PostgreSQL → **isso não acontece**

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

#### O problema que ele resolve

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

<a id="a-consequencia-do-mvcc-dead-tuples"></a>

### A consequência do MVCC — Dead tuples

Quando ocorre:

```sql
UPDATE users SET name = 'Ana Maria' WHERE id = 1;
```

O PostgreSQL:

**1.** cria nova versão da linha
**2.** marca a antiga com `xmax = tx_id`

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

<a id="e-agora-vacuum"></a>

### E agora? VACUUM!

#### O que o VACUUM realmente faz

Lembra das dead tuples?
Elas continuam dentro do arquivo físico da tabela (`heap file`).

O VACUUM:

**1.** varre página por página do heap
**2.** verifica visibility (usando xmin/xmax)
**3.** confirma que nenhuma transação precisa mais daquela versão
**4.** marca o espaço como reutilizável (free space map)

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
- quando passa do threshold → roda VACUUM + ANALYZE

Se ele estiver:

- desativado
- com `cost_limit` muito baixo
- com tabela muito grande
  → o banco lentamente degrada.

Este é o motivo n. 1 de "PostgreSQL ficou lento depois de meses em produção".

#### Transaction ID Wraparound (muito importante)

Cada transação recebe um `txid` de 32 bits.
Eles eventualmente **acabam**.

Se o banco não congelar linhas antigas via VACUUM:
o PostgreSQL entra em modo de proteção:

> ele começa a recusar escrita para evitar corrupção de dados

Ou seja:
**VACUUM não é otimização — é mecanismo de sobrevivência do banco**.

### Você vai escutar falar MUITO do WAL (Write-Ahead Log)

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

Sem WAL → corrupção lógica;

- dinheiro saiu de uma conta
- não entrou na outra conta

O banco ficaria inconsistente.

#### O que o PostgreSQL faz

Antes de alterar a tabela física (`heap`), ele escreve no WAL algo como:

```
tx 500:
page 123: old tuple → new tuple
page 456: old tuple → new tuple
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

> crash recovery → replay do WAL → restaura o banco ao último commit confirmado.

#### Consequência importante

Quando você vê:

> "PostgreSQL demorando no commit"

geralmente não é CPU nem query.

É:
**latência de fsync no disco do WAL**

Por isso SSD/NVMe impacta muito PostgreSQL.

<a id="e-o-que-sao-os-checkpoints"></a>

### E o que são os checkpoints?

Quando você executa:

```sql
UPDATE orders SET status = 'paid' WHERE id = 10;
```

O PostgreSQL:

**1.** escreve no WAL
**2.** altera a página na **RAM** (shared_buffers)

Ele **não grava imediatamente na tabela física**.

Isso é proposital — gravar em disco a cada query destruiria performance.
Essas páginas alteradas na memória são chamadas de _dirty pages_.

#### Então quando o banco grava de verdade no disco?

No **checkpoint**.

O processo de background (`checkpointer`) pega milhares de dirty pages e faz:

RAM → data files (heap/index)

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

<a id="o-que-significa-o-acronimo-acid"></a>

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

E o user **não existir** → <span style="color:red">erro</span>.

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

<a id="e-quais-sao-os-niveis-de-isolamento-no-postgresql"></a>

### E quais são os niveis de isolamento no PostgreSQL?

#### Os níveis no PostgreSQL

O padrão SQL define:
**1.** READ UNCOMMITTED
**2.** READ COMMITTED
**3.** REPEATABLE READ
**4.** SERIALIZABLE

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

Implementado com MVCC → não usa lock pesado.

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

Você executa novamente → aparece uma linha nova → phantom.

Isso acontece porque MVCC controla versões de linhas, **não o conjunto lógico do resultado**.

Somente **SERIALIZIBLE** impede.

**Impacto prático em backend**
Problemas clássicos:

- dupla cobrança
- estoque negativo
- criação duplicada de registro

APIs financeiras geralmente usam:
`SERIALIZIBLE` ou locking explicito.

<a id="o-que-e-deadlock-e-o-que-fazer-para-previnir"></a>

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

Isso é um ciclo de espera → deadlock

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

<a id="e-quais-sao-os-tipos-de-locks"></a>

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

Se a ordem de acesso for inconsistente → deadlock.
Se transações forem longas → lock rentention alto.
Se houver DDL em horário errado → freeze da aplicação.

<a id="e-como-o-postgresql-busca-os-dados-nas-tabelas"></a>

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

page 1 → page 2 → page 3 → ... → fim

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

**1.** consulta a B-tree
**2.** encontra ponteiro da linha
**3.** vai até o heap buscar a tupla

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

Alta seletividade → índice
Baixa seletividade → seq scan

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

<a id="como-o-postgresql-decide-executar-uma-query"></a>

### Como o PostgreSQL decide executar uma query

Toda vez que você roda:

```sql
SELECT * FROM orders WHERE status = 'paid';
```

o PostgreSQL **não sai executando imediatamente**.

Antes existe uma etapa crítica chamada:

> query planning (otimização de plano)

O banco precisa decidir:

- usar índice?
- fazer seq scan?
- nested loop ou hash join?
- qual ordem de joins?

E aqui está o ponto mais importante:

**o planner não olha a tabela para decidir**

Se ele fosse ler a tabela para decidir, já teria executado a query.

Então ele usa uma coisa intermediária: **estatísticas**.

#### Onde entram as estastísticas

O PostgreSQL mantém um resumo matemático das tabelas no catálogo interno: `pg_statistic`

Quem cria esse resumo é o `ANALYZE`.

O `ANALYZE` **não lê a tabela inteira**.
Ele faz amostragem e tenta responder:

> "Se alguém filtrar esta coluna, quantas linhas provavelmente vão aparecer?"

Ele coleta:

##### :one: MCV — Most Common Values

Valores mais frequentes.

| valor     | frequencia |
| --------- | ---------- |
| paid      | 92%        |
| pending   | 7%         |
| cancelled | 1%         |

---

##### :two: Histogram

Distribuição dos demais valores (usado para ranges)

Exemplo: `created_at > '2025-01-01'

---

##### :three: ndistinct

Qualidade estimada de valores únicos.

Importante para saber se índice vale a pena.

---

##### :four: null_frac

Percentual de NULLs

Afeta seletividade de filtros.

---

#### O que o planner realmente quer saber

Ele não quer saber **o valor**.
Ele quer saber quantas linhas a query vai retornar.

Isso se chama: **cardinality estimation**.

E isso decide o plano.

#### Como a estimativa decide o plano

```sql
SELECT * FROM orders WHERE status = 'paid' -- 92% da tabela
```

O planner calcula:

> "vou precisar ler quase a tabela inteira"

Então, **Seq Scan é mais barato que índice**.

Agora:

```sql
SELECT * FROM orders WHERE status = 'cancelled' -- 1% da tabela
```

> poucas linhas → índice compensa

Ou seja, **o PostgreSQL não escolhe índice por existir índice. Ele escolhe pelo número estimado de linhas.**

#### Onde entram os custos (cost)

Quando você roda:

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 10;
```

Ele não executa. Ele mostra a previsão:

```
Index Scan using idx_orders_user_id
(cost=0.42..8.44 rows=5 width=64)
```

Isso signifca:

- ele estima ~5 linhas
- custo baixo
- índice escolhido

O _cost_ é uma unidade interna baseada em:

- I/O esperado
- CPU esperado

Não é milissegundo.

#### O que acontece na execução real

Agora:

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 10;
```

A query executa de verdade:

```
(actual time=0.032..0.040 rows=1200 loops=1)
```

O banco esperava `rows=5`, mas encontrou `rows=1200`.

Aqui ocorreu erro de cardinalidade.
E isso é gravíssimo.
Porque o plano foi escolhido com base em 5 linhas, não em 1200.

Consequência:

- nested loop escolhido quando deveria ser hash join
- index scan quando deveria ser seq scan
- join order errado

Agora você tem slow query **mesmo com índice correto**.

#### Por que a estimativa ficou errada

Porque o planner confia no `pg_statistic`. E ele depende do `ANALYZE`.

Se a distribuição mudou:

- muitos inserts
- mudança de comportamento do sistema
- campnha
- feature nova

as estastísticas ficam obsoletas.

Então o banco está tomando decisão correta para um banco que **não existe mais**.

#### Quem atualiza automaticamente

O `autovacuum` faz duas coisas diferentes:

```
VACUUM → limpa dead tuples (MVCC)
ANALYZE → atualiza estatísticas (planner)
```

Ou seja:

- `VACUUM` mantém armazenamento saudável
- `ANALYZE` mantém decisões inteligentes

Sem `ANALYZE` o PostgreSQL fica "cego" para otimização.

#### Como você diagnostíca isso

Você usa `EXPLAIN ANALYZE`.

Ele compara:

| estimado | real      |
| -------- | --------- |
| rows=5   | rows=1200 |

Diferença grande → problema de estatística.

Solução:

```sql
 ANALYZE orders
```

ou ajustar `autovacuum`.

Fluxo real dentro do PostgreSQL:

1. Dados mudam.
2. Estatísticas ficam antigas.
3. Planner estima errado.
4. Plano errado é escolhido.
5. Query fica lenta.
6. `EXPLAIN ANALYZE` revela diferença.
7. `ANALYZE` corrige.

<a id="indices-do-postgresql"></a>

### Índices do PostgreSQL

:one: **B-Tree (90% dos casos)**

```sql
CREATE index (idx_users_email) ON users(email)
```

Otimiza:

```
WHERE email = ?
WHERE created_at > ?
ORDER BY created_at
```

Ele funciona bem porque mantém ordenação.

Use para:

- PK
- FK
- datas
- buscas exatas

:two: **Hash**

```sql
CREATE INDEX idx_users_email_hash ON users USING HASH(email)
```

Só acelera:

```
WHERE email = ?
```

Não serve para range nem ordenação.
Na prática quase nunca compensa usar em vez de B-Tree.

:three: **GIN (muito importante em APIs modernas)**

Exemplo JSONB:

```sql
CREATE INDEX idx_products_attrs ON products USING GIN(attributes)
```

Consulta:

```sql
WHERE attributes @> '{"color": "red"}'
```

Sem GIN → full table scan.

Também usado para:

- tags
- arrays
- busca textual (`to_tsvector`)

:four: **GiST**
Usado quando você não quer igualdade, mas **proximidade**.

Exemplo:

- distância geográfica
- ranges de data
- similaridade

Comum em PostGIS
"restaurantes perto de mim".

:five: **BRIN (muito subestimado)**
Para tabela enorme:

```sql
logs (timestamp, ...)
```

Inserções sempre crescentes.

```sql
CREATE INDEX idx_logs_time ON logs USING BRIN(timestamp)
```

Ele não indexa cada linha
Ele indexa **blocos de páginas**.

Resultado:

- índice minúsculo
- escrita barata
- ótimo para time-series

#### Impacto em produção

Índice também tem custo:

Cada `INSERT/UPDATE`:

- atualiza todos os índices
- gera WAL
- aumenta checkpoint

Índice demais → write performance piora.

#### O comportamento normal de um índice

Mesmo quando um índice é usado:

```sql
SELECT email FROM users WHERE email = 'ana@email.com';
```

O PostgreSQL:

**1.** consulta a B-Tree
**2.** encontra o ponteiro (TID)
**3.** vai até a tabela (heap) buscar a linha

Ou seja:

> Índice não guarda a linha, só o endereço dela.

Isso se chama:
**heap fetch**

E isso custa caro.

#### Quando ocorre Index Only Scan

Se você criar:

```sql
CREATE INDEX idx_users_email ON users(email);
```

E a query pedir **apenas a coluna do índice**:

```sql
SELECT email FROM users WHERE email = 'ana@email.com';
```

O PostgreSQL _pode_ evitar acessar a tabela.

Plano esperado:

```
Index Only Scan using idx_users_email
```

Mas aqui entra o MVCC.

#### O problema da visibilidade

O índice não sabe se a linha ainda é visível para sua transação.
(lembrando: múltiplas versões da mesma linha existem)

Então o PostgreSQL precisa confirmar:

> essa tupla não foi deletada ou atualizada depois?

Quem fornece essa informação é o **visibility map**.

Ele marca páginas do heap como:

- todas as linhas visíveis
- seguras para leitura direta

#### Quem atualiza o visibility map?

**VACUUM**.

Se o autovacuum não estiver rodando bem:

o plano vira:

```
Index Scan + Heap Fetch
```

Ou seja:
o banco volta a acessar a tabela → perde performance.

Por isso existe um fenômeno real:

> banco com índice correto mas SELECT lento

Frequentemente é **autovacuum atrasado**.

#### Impacto prático

Index Only Scan:

- reduz I/O
- melhora cache hit
- melhora latência

Mas depende diretamente de:

- MVCC
- dead tuples
- VACUUM

É por isso que PostgreSQL não é só "criar índice".

#### Partial Index (muito útil em produção)

Imagine tabela:

`orders`

- 95% = `delivered`
- 5% = `open`

API mais comum:

```sql
SELECT * FROM orders
WHERE user_id = 10 AND status = 'open';
```

Se você fizer índice normal:

```sql
CREATE INDEX idx_orders_status_user ON orders(status, user_id);
```

O índice fica enorme e cada `INSERT/UPDATE` paga custo alto.

Agora:

```sql
CREATE INDEX idx_orders_open_user
ON orders(user_id)
WHERE status = 'open';
```

Você indexa só o que realmente consulta.

Quando uma linha deixa de satisfazer a condição do partial index, o PostgreSQL remove a entrada correspondente do índice.

Isso ocorre durante o UPDATE, como parte da manutenção normal de índices.

Internamente:

- o UPDATE cria nova versão da tupla (MVCC)
- a versão antiga vira dead tuple
- o índice é atualizado conforme a nova versão atende (ou não) ao predicado

O índice continua existindo; apenas aquela linha deixa de fazer parte dele.

Benefícios:

- índice pequeno
- menos WAL
- menos checkpoint
- mais cache

Muito usado em:

- filas
- soft delte (`deleted_at IS NULL`)
- registros ativos

#### Composite Index (ordem crítica)

B-Tree é ordenada.

Índice:

```sql
(user_id, created_at)
```

Fisicamente fica:

```
user_id 1 → todas as datas
user_id 2 → todas as datas
user_id 3 → todas as datas
```

Então funciona bem:

```sql
WHERE user_id = 10
WHERE user_id = 10 AND created_at > now()-interval '7 days'
ORDER BY created_at
```

Mas não:

```sql
WHERE created_at > now()-interval '7 days'
```

Porque o banco não consegue pular direto — teria que percorrer todo o índice.

#### Regra prática (muito perguntada)

Coluna mais seletiva primeiro?
**Não exatamente.**

A regra real:

> coloque primeiro a coluna mais usada como filtro inicial das queries

#### Impacto real em backend

Índice composto errado gera:

- Seq Scan inesperado
- alto CPU
- aumento de latência
- pool de conexões saturando

E normalmente o dev diz:

> "mas eu já criei índice..."

<a id="paginacao-offset-x-keyset"></a>

### Paginação: OFFSET x Keyset

#### OFFSET pagination

Exemplo clássico:

```sql
SELECT * FROM orders
ORDER BY id
LIMIT 20 OFFSET 100000;
```

O que o PostgreSQL faz:

Ele **não pula\*\*** 100 mil linhas.

Ele:

1. percorre 100000 linhas
2. descarta
3. só então retorna 20

Ou seja:

- lê indice
- faz heap fetch
- joga fora tudo

Página 1 → rápida
Página 5000 → extremamente lenta

E piora:
cada usuário acessando páginas altas repete esse custo.

#### Outro problema (concorrência)

Se novas linhas forem inseridas:

- usuário pode ver registros duplicados
- ou pular registros

OFFSET não é determinístico sob escrita concorrente.

#### Keyset / Cursor pagination

Você usa a última chave retornada:

```sql
SELECT * FROM orders
WHERE id > 100000
ORDER BY id
LIMIT 20;
```

Agora o banco:

- vai direto no índice
- continua a partir d oponto
- não precisa percorrer o passado

Complexidade:

**O(log n)** ao invés de **O(n)**.

#### Por que escala muito melhor

O índice B-Tree já é ordenado.

O banco só faz:

> continuar a navegação da árvore

Nenhum descarte de linhas.

#### Impacto real em backend

OFFSET grande gera:

- seq scan parcial
- alto tempo de query
- pool de conexões ocupado
- aumento de latência global da API

Sintoma típico:

> endpoint de listagem derruba o sistema em horário de pico.

<a id="pool-de-conexoes"></a>

### Pool de conexões

Aqui está a parte crítica:

PostgreSQL **não é thread-based como MySQL**.

Ele funciona assim:

```
1 conexão TCP → 1 processo do PostgreSQL
```

Se você tiver:

- 300 conexões
- você tem 300 processos no servidor

Cada processo:

- tem memória própria
- participa do scheduler do Linux
- mantém transação e snapshot MVCC

#### Por que Node.js agrava isso

Node.js é altamente concorrente:

- 1 servidor
- milhares de requests simultâneos

Se você fizer:

```js
new Client().connect(); // por request
```

Cada request abre uma conexão nova.

Resultado real em produção:

- 800 conexões abertas
- banco fica lento
- CPU do DB 100%
- queries simples demoram

Não é a query
É o **gerenciamento de processos do banco**.

#### O que o pool resolve

O pool faz:

```
1000 requisições web
↓
20 conexões reais no banco
↓
fila interna na aplicação
```

Ou seja:
Você limita a concorrência no ponto correto: **antes do banco**.

Isso melhora:

- latência global
- estabilidade
- uso de memória
- lock contetion

#### PgBouncer (muito perguntado)

**O problema real que existe sem PgBouncer**

Você tem:

- API Node.js
- 4 pods
- cada pod com pool = 20 conexões

Total:

```
4 x 20 = 80 conexões no PostgreSQL
```

Até aqui tudo bem.

Agora acontece algo normal em produção:

- pico de tráfego
- autoscaling

Você sobe pra **20 pods**.

Agora:

```
20 x 20 = 400 conexões abertas no banco
```

No PostgreSQL isso é grave porque:

> conexão não é só um socket — é um processo inteiro do servidor.

O banco agora tem 400 processos competindo por:

- CPU
- memória
- locks
- buffer cache

Mesmo que as queries sejam leves, o banco degrada.
Ele passa mais tempo **gerenciando conexões do que executando SQL**.

**O que o PGBouncer faz (de forma concreta)**
Ele vira um **multiplexador de conexões**.

Arquitetura:

```
Clientes (Node)
        ↓
PgBouncer
        ↓
PostgreSQL
```

Agora:

- Node pode abrir 400 conexões
- mas o PgBouncer mantém **ex.: 30 conexões reais no banco**

Ou seja:

```
400 sessões lógicas → 30 sessões físicas
```

O segredo:

O PgBouncer **não mantém a conexão presa ao cliente o tempo inteiro**.
Ele só usa uma conexão real **durante a execução da query**.

Terminou a query:

- a conexão volta ao pool
- outro cliente usa a mesma

Isso funciona porque a maioria das APIs faz:

```
query curta → resposta → acabou
```

Você não precisa de uma sessão dedicada permanente

**Por que isso é especialmente importante no PostgreSQL**
Porque o PostgreSQL:

- não foi projetado para milhares de conexões
- foi projetado para **poucas conexões ativas bem utilizadas**

Na prática:

| Conexões | Efeito                |
| -------- | --------------------- |
| 20-50    | ótimo                 |
| 100-200  | aceitável             |
| 400+     | degradação séria      |
| 1000+    | quase sempre instável |

Não é CPU da query
É **overhead de processos + memória + locks + snapshots MVCC**.

**O modelo interno do PostgreSQL**
O PostgreSQL usa:

> **process-per-connection**

Cada conexão cria um processo do sistema operacional (`postgres: backend`).

Então 1000 conexões = **1000 processos reais no Linux**.

Não são threads leves.
São processos completos:

- memória própria
- pilha
- descritores de arquivo
- contexto de execução
- snapshot do MVCC

**O que acontece numa máquina poderosa?**

Imagine:

- 32 cores
- 128 GB RAM
- NVMe rápido

Você pode pensar:

> aguenta 1000 conexões fácil

Na prática não.

Porque o gargalo vira outro: **agendamento do sistema operacional (scheduler)**.

O Linux precisa:

- pausar processos
- restaurar contexto
- alternar CPU (context switch)

Com centenas de backends:

O banco passa muito tempo fazendo:
**gerenciamento de concorrência**, não execução de query.

Isso gera:

- aumento de latência
- queda de throughput total

Isso é chamado:

> connection trashing

**O segundo problema (mais importante ainda)**
Cada backend mantém:

- memória local
- estruturas de lock
- snapshot de transações

Especialmente:
**MVCC snapshot tracking**

O PostgreSQL precisa saber:

> qual a transação mais antiga ainda ativa?

Porque ele não pode remover versões que ainda podem ser visíveis.

Com muitas conexões abertas (mesmo ociosas), o banco assume que alguma pode precisar de dados antigos.

Resultado:

VACUUM não consegue limpar.

Consequência direta

- dead tuples acumulam
- tabela cresce
- índice cresce
- cache piora
- queries ficam mais lentas

Isso acontece mesmo com CPU sobrando.

**Então mais hardware resolve?**
Ajuda, mas não resolve.

Você ganha:

- mais buffer cache
- mais I/O

Mas não elimina:

- context switching
- lock tables maiores
- MVCC visibility tracking
- gerenciamento de snapshots

Por isso todos os guias oficiais e DBAs dizem:

> PostgreSQL escala melhor em **conexões ativas pequenas + queries rápidas**, não em muitas conexões simultâneas.

**Número típico recomendado**
Produção comum:

| Tipo de servidor | Conexões ativas ideais |
| ---------------- | ---------------------- |
| pequeno          | 20-50                  |
| médio            | 50-100                 |
| grande           | 100-200                |

Acima disso normalmente:
você não ganha throughput — você perde.

Por isso PgBouncer não é otimização.
Ele é **parte da arquitetura padrão de produção** em PostgreSQL.

#### Diferença entre pool de aplicação e PgBouncer

##### Pool do Node (`pg.Pool`)

Controla concorrência **dentro de um único processo Node**.

Ele não sabe:

- quantos pods existem
- quantos serviços existem
- quantos workers existem

Cada instância cria o próprio pool → o total explode.

##### PgBouncer

Controla concorrência **global do banco**.

Ele vira o "porteiro":

> "O banco só vai trabalhar com 30 pessoas ao mesmo tempo, não importa quantos clientes cheguem."

Isso estabeliza:

- latência
- VACUUM
- checkpoints
- locks

<a id="o-problema-de-transacoes-longas"></a>

### O problema de transações longas

Lembre do MVCC:

cada transação enxerga o banco como ele estava **no momento do BEGIN**.

Exemplo:

Transação A:

```sql
BEGIN;
SELECT * FROM orders; -- snapshot tirado agora
```

Ela fica aberta (ex.: usuário abriu página e deixou o navegador parado)

Enquanto isso o sistema continua:

```sql
UPDATE orders ...
DELETE orders ...
INSERT orders ...
```

O banco vai criando:

- novas versões
- e versões antigas virando dead tuples

O banco vai criando:

- novas versões
- e versões antigas virando dead tuples

Normalmente o VACUUM limparia.

Mas aqui está o ponto crítico:

> O PostgreSQL não pode apagar uma linha antiga se ainda existir uma transação que possa enxergá-la.

A transação A ainda pode ver o banco antigo.

Então o VACUUM fica impedido.

#### O efeito dominó

Quanto mais tempo a transação fica aberta:

1. mais versões antigas acumulam
2. tabela cresce
3. índice cresce
4. cache piora
5. queries ficam mais lentas
6. checkpoint mais pesado
7. WAL cresce

Você começa a observar:

> "o banco foi ficando lento ao longo do dia"

Muito comum em APIs que:

- abrem transação
- chamam serviço externo
- aguardam fila
- fazem upload
- ou usam ORM mal configurado

#### O caso clássico em Node.js

Erro típico:

```js
BEGIN
SELECT ...
await chamarAPIexterna() // 3 segundos
UPDATE ...
COMMIT
```

Durante esses 3 segundos:
o banco inteiro fica impedido de limpar versões antigas.

Com várias requisições → degradação global.

##### O pior cenário

Se durar horas:

o banco se aproxima do:
**transaction id wraparound**

O PostgreSQL entra em modo de proteção:
ele começa a bloquear writes para evitar corrupção.

Ou seja:
uma única conexão esquecida pode degradar todo o banco.

<a id="replicas"></a>

### Replicas

Lembra do WAL?

Toda alteração no PostgreSQL primeiro vai para o WAL

O primário faz:

```
UPDATE → grava WAL → commit
```

Na replicação:

o primário também **envia o WAL via rede** para outro servidor.

O standby recebe:

```
recebe WAL → reexecuta alterações → atualiza seus data files
```

Ele não executa SQL da aplicação.
Ele apenas **reaplica as mudanças**.

Isso se chama:

> physical streaming replication

#### O que é uma read replica

Você pode conectar sua aplicação nela:

- SELECT permitido
- UPDATE/INSERT/DELETE proibido

Arquitetura:

```
API
 ├── writes → primary
 └── reads → replica
```

Muito usado em:

- dashboards
- relatórios
- listagens grandes
- analytics

#### O que ela resolve

1. Alta disponibilidade
   Se o primário cair → promove standby (failover)
2. Escala leitura
   Você tira carga de SELECT pesado do primário

#### O que ele NÃO resolve (pegadinha clássica)

Ela não ajuda em:

- UPDATE lento
- locks
- deadlocks
- checkpoint
- VACUUM
- escrita intensa

Porque **todo write ainda acontece no primário**.

#### Replication Lag

Como o WAL precisa:

- ser enviado
- aplicado

pode existir atraso.

O usuário escreve:

```
POST /order
```

logo depois:

```
GET /order
```

Se o GET bater na replica → pode não encontrar.

Isso chama:

> eventual consistency

Aplicações precisam lidar com isso.

<a id="failover"></a>

### Failover

#### O cenário

Arquitetura:

```
Primary → envia WAL → Standby
```

Se o primary morre:

- standby ainda está rodando
- mas é read-only

Failover significa:

> promover standby a primary

Ele passa a aceitar writes.

#### O ponto crítico: risco de perda de dados

##### Replicação assíncrona (default)

Fluxo:

```
commit no primary
↓
responde cliente
↓
envia WAL para replica
```

Se o primary cair entre:

- commit
- envio do WAL

O standby nunca recebeu aquela transação.

Se promovido:
→ você perdeu dados confirmados ao cliente.

Isso é chamado:

> window of data loss

##### Replicação síncrona

Fluxo

```
commit
↓
envia WAL
↓
replica confirma recebimento
↓
primary responde cliente
```

Agora não há perda (no limite de nós sincronizados)
Mas custo:

- commit mais lento
- depende da latência da rede
- risco de bloquear se replica ficar indisponível

#### Outro ponto crítico

Failover não é automático por padrão

Você precisa:

- monitoramento
- orquestrador
- reconfiguração de DNS ou load balancer

Senão a aplicação continua tentando conectar no primary morto.

#### Conclusão arquitetural

| Sistema          | Estratégia |
| ---------------- | ---------- |
| financeiro       | síncrona   |
| e-commerce comum | assíncrona |
| analytics        | assíncrona |

Sempre há trade-off entre **durabilidade vs latência**

<a id="migrations"></a>

### Migrations

#### O que derruba a aplicação

Você executa

```sql
ALTER TABLE users ADD COLUMN age INT;
```

O PostgreSQL precisa garantir consistência estrutural.

Ele pega:

> `ACCESS EXCLUSIVE LOCK`

Isso significa:

- ninguém lê
- ninguém escreve
- ninguém atualiza

Não é só a tabela:
as conexões ficam esperando.

No backend você vê:

- pool esgotado
- requisições acumulando
- CPU da API sobe
- usuários recebem timeout

A aplicação não caiu
Ela **ficou bloqueada pelo banco**.

#### Outro exemplo comum

Criar índice:

```sql
CREATE INDEX idx_orders_user ON orders(user_id)
```

O PostgreSQL:

- lê a tabela inteira
- mantém lock de escrita

Em tabela grande → minutos de bloqueio.

#### Forma correta

```sql
CREATE INDEX CONCURRENTLY idx_orders_user ON orders(user_id);
```

Agora:

- tabela continua operando
- banco cria índice em background
- sem travar a API

#### Mudança de coluna (o mais perigoso)

```sql
ALTER TABLE orders ALTER COLUMN price TYPE numeric(12,2)
```

Isso força **table rewrite**:

- copia toda a tabela
- bloqueia
- WAL gigante
- checkpoint pesado

Pode travar produção por horas.

#### Estratégia real de zero-downtime

Você faz em fases:

1. adiciona nova coluna nullable
2. código passa a escrever nas duas
3. backfill em batches
4. muda leitura
5. remove coluna antiga

Compatibilidade dupla temporária

#### Por que isso é crítico

A maioria dos incidentes de banco em produção não é query lenta é **migration bloqueante**.

<a id="prepared-statements"></a>

### Prepared Statements

Quando você executa normalmente:

```sql
SELECT * FROM orders WHERE user_id = 10;
```

O PostgreSQL faz 3 etapas toda vez:

1. **parse** - entende o SQL
2. **plan** - decide índice/scan/join
3. **execute**

Isso custa CPU.

#### O prepared statement

Você prepara uma vez:

```sql
PREPARE get_orders(int) AS
SELECT * FROM orders WHERE user_id = $1;
```

Depois:

```sql
EXECUTE get_orders(10);
EXECUTE get_orders(20);
EXECUTE get_orders(30);
```

Agora o banco:

- não reinterpreta SQL
- não recalcula plano toda vez

Ganho real:
APIs com milhares de requests por segundo.

Drivers como `pg` do Node já fazem isso implicitamente.

#### O problema com pool / PgBouncer

Prepared statement vive **dentro da conexão**.

Mas com PgBouncer em _transaction pooling_:

request A:

```
usa conexão 3 → prepara statement
```

request B:

```
vai para conexão 7 → statement não existe
```

Erro clássico:

```
prepared statement "S_1" does not exist
```

Ou pior, plano cacheado para um valor raro.

Exemplo:
`user_id = 1` → 500k linhas
`user_id = 9999` → 2 linhas

O planner escolhe plano baseado no primeiro uso e reutiliza para todos → query fica lenta.

Isso se chama **parameter sniffing / generic plan problem**.

#### Quando usar

Bom:

- queries muito frequentes
- lookup por id

Cuidado:

- queries altamente variáveis
- PgBouncer transaction pooling

Soluções comuns:

- desabilitar prepare no driver
- usar PgBouncer session mode
- `prepareThreshold` tuning

<a id="o-problema-de-n1-queries"></a>

### O problema de N+1 queries

Você faz:

```sql
SELECT * FROM users;
```

A aplicação recebe 100 usuários.

O código então faz:

```sql
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 2;
SELECT * FROM orders WHERE user_id = 3;
...
```

Resultado:

**1 + 100 = 101 queries**

O banco não sofre por uma query pesada
Ele sofre por **muitas queries pequenas simultâneas**.

#### Por que ORMs causam

Exemplo típico (pseudo-ORM)

```js
const users = await User.findAll();

for (const user of users) {
  await user.getOrders();
}
```

Isso ativa lazy loading.

Cada iteração:

- abre transação
- pega conexão
- cria snapshot
- executa query
- libera

Sob 200 requisições recorrentes:

```
200 requests x 101 queries = 20.200 queries quase simultâneas
```

Agora o problema deixa de ser SQL.

Vira:

- pool esgotado
- context switching
- WAL flush frequente
- autovacuum atrasado

#### Forma correta

Banco foi feito para resolver reclação:

```sql
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;
```

Ou:

```sql
SELECT * FROM orders
WHERE user_id IN (1,2,3,4,...);
```

Uma query mais pesada é melhor do que centenas pequenas.

#### Impacto real em produção

Sintomas clássicos:

- CPU do banco sobe
- queries simples ficam lentas
- número de conexões aumenta
- "mas a query individual é rápida"

O problema é **multiplicação concorrente**, não custo unitário.

<a id="idempotencia-no-postgresql"></a>

### Idempotência no PostgreSQL

---

#### Estratégia 1 — Idempotency Key

Cliente envia:

```json
{
  "amount": 100,
  "idempotency_key": "abc-123"
}
```

No banco:

```sql
CREATE UNIQUE INDEX idx_idempotency
ON payments(idempotency_key)
```

Inserção:

```sql
INSERT INTO payments (...)
VALUES (...)
ON CONFLICT (idempotency_key)
DO NOTHING;
```

Se já existir:

- a operação não duplica
- você retorna o pagamento existente

---

#### Estratégia 2 — Unique constraint lógica

Exemplo:

```sql
CREATE UNIQUE INDEX idx_unique_order
ON payments(order_id)
```

Agora duas cobranças para o mesmo pedido:

- segunda falha
- banco garante integridade

---

#### Estratégia 3 — Lock explícito

Para casos complexos:

```sql
SELECT ... FOR UPDATE;
```

Ou advisory lock.

---

#### O ponto importante

Idempotência **não é lógica de aplicação**

É restrição garantida pelo banco.

Se depender só de:

```sql
UPDATE ... WHERE status = 'pending'
```

sob concorrência alta você pode ter:

- race condition
- dupla execução
- inconsistência

<a id="fila-usando-postgresql"></a>

### Fila usando PostgreSQL

Tabela:

```
jobs
(id, payload, status, created_at)
```

Status:

- pending
- processing
- done
- failed

#### O erro comum

Workers fazem:

```sql
SELECT * FROM jobs WHERE status='pending' LIMIT 1;
```

Problema:

Dois workers podem pegar o mesmo job simultaneamente.

Race condition.

#### A solução do PostgreSQL

Worker:

```sql
BEGIN;

SELECT id, payload
FROM jobs
WHERE status='pending'
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

O que acontece:

- a linha selecionada recebe **row lock**.
- outros workers não esperam
- eles pulam (skip) e pegam outro job

Agora:

```sql
UPDATE jobs
SET status='processing'
WHERE id = ?;

COMMIT;
```

O lock impede duplicação

#### Por que funciona

`FOR UPDATE`:
bloqueia linha

`SKIP LOCKED`:
evita fila de espera

Sem `SKIP LOCKED`:
os workers ficariam bloqueados uns nos outros.

#### Benefícios

- transacional
- sem duplicação
- sem serviço externo
- respeita ACID

#### Limitações

Não substitui Kafka/RabbitMQ quando:

- milhões de jobs/minuto
  processamento distribuido pesado

Mas é excelente para:

- emails
- webhooks
- retry jobs
- processamento interno

<a id="o-que-e-pg_stat_activity-e-pg_stat_statements-e-qual-sua-importancia"></a>

### O que é `pg_stat_activity` e `pg_stat_statements` e qual sua importância?

Quando alguém diz:

> "o banco está lento"

Existem duas perguntas diferentes:

1. O que está acontecendo **agora**?
2. Qual query está custando mais ao longo do tempo?

São ferramentas diferentes.

#### :one: pg_stat_activity (tempo real)

```sql
SELECT * FROM pg_stat_activity;
```

Você vê:

- pid
- state
- query
- query_start
- wait_event
- xact_start

Exemplo útil:

```sql
SELECT pid, state, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active';
```

Isso revela:

- queries rodando há 5 minutos
- transações abertas
- bloqueios

Você também pode identificar:

```
idle in transaction
```

Isso é perigoso:
→ transação aberta segurando snapshot → impedindo VACUUM.

#### :two: pg_stat_statements (histórico agregado)

Precisa estar habilitado (`shared_preload_libraries`).

Consulta típica:

```sql
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

Isso mostra:

- quais queries mais consumiram tempo total
- não apenas lentas, mas frequentes

Exemplo real:

| query             | calls | mean_exec_time |
| ----------------- | ----- | -------------- |
| SELECT user by id | 500k  | 1ms            |
| SELECT order list | 2k    | 400ms          |

Talvez a primeira seja mais problemática por volume

#### Como usar juntos em incidente real

Produção lenta.

##### Passo 1

Ver se há bloqueio

```sql
SELECT * FROM pg_stat_activity WHERE wait_event IS NOT NULL;
```

##### Passo 2

Ver long transactions:

```sql
SELECT pid, now() - xact_start
FROM pg_stat_activity
WHERE state = 'active';
```

##### Passo 3

Ver queries mais custosas historicamente:

```sql
SELECT query, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC;
```

#### Diferença essencial

| Ferramenta         | Serve para                   |
| ------------------ | ---------------------------- |
| pg_stat_activity   | diagnóstico imediato         |
| pg_stat_statements | tuning e otimização continua |

#### Impacto arquitetural

Sem `pg_stat_statements`, você otimiza "no escuro".
Sem `pg_stat_activity`, você não resolve indicentes ao vivo.

<a id="o-que-e-pitr-point-in-time-recovery"></a>

### O que é PITR (Point-in-Time Recovery)?

Lembra do WAL?

Toda mudança no banco vai primeiro para o WAL:

```
INSERT / UPDATE / DELTE → WAL → data files
```

Ou seja, o WAL é literalmente um **registro cronológico de tudo que aconteceu no banco**.

#### O problema do backup comum

Você faz backup às 02:00.

Às 15 alguém executa:

```sql
DELETE FROM users;
```

Agora você só tem:

- backup de 02:00
- dados destruídos às 15:00

Restaurar o backup fará você perder **13 horas de dados legítimos**.

#### O que o PITR faz

Você também arquiva os WALs continuamente.

Então você tem:

```
backup base → WAL 02:01 → WAL 02:02 → ... → WAL 14:59 → WAL 15:00
```

Agora o PostgreSQL pode:

1. restaurar o backup das 02:00
2. reexecutar (replay) todos os WALs
3. **parar exatamente às 14:59:59**

Ou seja, você volta o banco para antes do erro humano. Isso é literalmente viagem no tempo do banco.

#### Como você escolhe o ponto

Você pode parar por:

- timestamp
- transaction id
- LSN (log sequence number)

Exemplo conceitual:

```
recovery_target_time = '2026-03-01 14:59:59'
```

#### Por que isso é crítico em produção

Incidentes reais:

- `DELETE sem WHERE`
- bug em código
- migration errada
- importação corrompida
- ransomware

Sem PITR:
downtime + perda massiva de dados.

Com PITR:
recuperação cirúrgica.

<a id="quando-usar-replicacao-fisica-e-quando-usar-replicacao-logica"></a>

### Quando usar replicação física e quando usar replicação lógica

Lembra do WAL?

Ele registra **alterações de páginas do disco**, não queries SQL.

Exemplo interno (conceitual):

```
página 812 → linha 4 mudou de X para Y
```

Isso é o que a replicação física envia.

#### Replicação física (streaming replication)

O primário envia WAL → standby reaplica.

A réplica não entende negócio, só reproduz o armazenamento.

Consequências:

- estrutura idêntica
- índices idênticos
- mesmas tabelas
- não pode alterar schema
- não pode escrever

É literalmente um "espelho vivo" do banco.

Por isso serve para:

- failover
- leitura

Mas não para integração.

#### Replicação lógica

Aqui o PostgreSQL decodifica o WAL.

Ele transforma:

```
page change
```

em:

```sql
INSERT INTO orders ...
UPDATE users ...
DELETE FROM payments ...
```

Agora você pode:

- escolher tabelas
- enviar para outro PostgreSQL
- enviar para Kafka
- alimentar data warehouse

Isso se chama **CDC (Change Data Capture)**

#### Exemplo prático

Você quer mandar pedidos para um sistema de BI sem impactar produção.

Com replicação física teria cópia inteira do banco.

Com lógica, replica só `orders`.

#### Migração sem downtime (caso clássico)

Você quer trocar:

PostgreSQL antigo → novo cluster

Você:

1. cria novo banco
2. ativa replicação lógica
3. dados sincronizam continuamente
4. troca aplicação

Sem parar o sistema.

#### Trade-offs

| Física      | Lógica      |
| ----------- | ----------- |
| rápida      | mais pesada |
| simples     | complexa    |
| consistente | flexível    |
| tudo        | seletiva    |
| HA          | integração  |

#### Conexão com WAL

- física → replay do WAL
- lógica → decodificação do WAL

Mesma fonte, usos diferentes

<a id="contencao-de-locks"></a>

### Contenção de locks

Imagine 500 requisições simultâneas fazendo:

```sql
UPDATE accounts SET balance = balance - 1 WHERE id = 1;
```

Cada transação:

1. cria snapshot
2. bloqueia a mesma linha (row lock)
3. escreve no WAL
4. faz commit (fsync)
5. libera lock

Mesmo sendo rápidas (2–3ms), elas formam fila de lock na mesma linha

Não é deadlock.
É serialização forçada.

#### Contenção invisível (mais comum ainda)

Mesmo que atualizem linhas diferentes:

```sql
UPDATE orders SET status = 'paid' WHERE id = $1;
```

Ainda competem por:

#### 1️⃣ WAL flush

Commit precisa garantir WAL persistido no disco.

Se 500 commits simultâneos → todos esperam o disco.

---

#### 2️⃣ Index page contention

Se muitos inserts usam o mesmo índice crescente:

**inserções no final da B-Tree**

Todos disputam a mesma página do índice.

---

#### 3️⃣ LWLocks (locks internos)

PostgreSQL tem locks internos para:

- buffer cache
- freelist
- WAL buffers
- clog

Esses não aparecem como row locks.

Mas geram espera.

---

#### 4️⃣ Context switching

500 conexões = 500 processos

Mesmo com CPU sobrando:

- scheduler troca contexto
- cache de CPU invalida
- overhead cresce

Throughput começa a cair.

---

#### O fenômeno real

Se você aumentar conexões:

| Conexões | Throughput    |
| -------- | ------------- |
| 10       | cresce        |
| 30       | cresce        |
| 80       | ótimo         |
| 150      | estabiliza    |
| 300      | começa a cair |
| 600      | degrada forte |

Isso se chama **_contention collapse_**

#### Por que isso acontece mesmo com transações curtas?

Porque:

- elas ainda fazem commit
- commit exige WAL flush
- flush é serializado
- lock release ainda é sequencial
- CPU ainda troca contexto

Mesmo 2ms × 500 simultâneas → gargalo.

#### Solução arquitetural

- limitar pool
- usar PgBouncer
- batch writes
- reduzir commits desnecessários
- evitar hot rows
- shardear se necessário

#### Conclusão importante

PostgreSQL escala melhor com:

> poucas conexões bem utilizadas
> do que muitas conexões concorrendo

Concorrência excessiva reduz eficiência.

### O que é Hot Row Contention?

Imagine uma tabela:

```
inventory
(product_id, stock)
```

API de compra:

```sql
UPDATE inventory
SET stock = stock - 1
WHERE product_id = 10;
```

Agora ocorre uma promoção

500 usuários clicam em **comprar** ao mesmo tempo.

#### O que acontece dentro do PostgreSQL

Primeira transação

- pega row lock na linha `product_id = 10`

As outras 499:

- não podem atualizar
- ficam esperando

Elas não falham.
Elas entram em **fila de lock**.

Resultado:

| tempo  | efeito            |
| ------ | ----------------- |
| início | rápido            |
| pico   | latência cresce   |
| depois | pool esgota       |
| final  | API lenta/timeout |

O banco não está lento por CPU
Está lento por **serialização forçada**.

#### Como identificar

Rodando:

```sql
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL;
```

Você verá várias sessões esperando e a mesma query:

```sql
UPDATE inventory SET stock = stock - 1 WHERE product_id = $1
```

Isso é assinatura clássica de hot row.

#### Por que MVCC não resolve

MVCC permite:

SELECT não bloquear UPDATE.

Mas UPDATE x UPDATE na mesma linha
→ precisa serializar.

Não tem como duas versões modificarem o mesmo registro simultaneamente.

#### Soluções reais

##### :x: ERRADO (modelo ingênuo)

contador central:

```
likes = likes + 1
saldo = saldo - 1
estoque = estoque - 1
```

---

##### :heavy_check_mark: SOLUÇÃO 1 — tabela de eventos

Ao invés de atualizar:

```sql
INSERT INTO stock_movements(product_id, delta) VALUES (10, -1);
```

Depois agrega:

```sql
SELECT SUM(delta) FROM stock_movements WHERE product_id = 10;
```

Agora:
cada compra escreve em linha diferente → paralelizável

---

##### :heavy_check_mark: SOLUÇÃO 2 — sharding lógico

Em vez de 1 contador:

```
counter_1
counter_2
counter_3
counter_4
```

Cada request usa um shard aleatório
Depois soma

---

##### :heavy_check_mark: SOLUÇÃO 3 — fila

- grava pedido
- worker sequencial desconta estoque

Remove concorrência

---

##### :heavy_check_mark: SOLUÇÃO 4 — cache

Redis INCR/DECR para contadores de alta frequência

---

#### Conclusão importante

Hot row contention é um problema **de modelagem de dados**, não de índice nem hardware

Você não resolve aumentando CPU.
Você resolve **mudando o padrão de escrita**.

### O que é starvation?

Vamos usar a fila de jobs:

```sql
SELECT *
FROM jobs
WHERE status = 'pending'
LIMIT 1
FOR UPDATE;
```

Você tem 10 workers.

#### O que você espera

Cada worker pega um job diferente.

#### O que pode acontecer

O banco não garante **justiça de agendamento**

Cenário:

1. Worker A tenta lockar linha 1
2. Worker B chega logo depois → consegue linha 2
3. WOrker C chega → linha 3
4. Worker D → linha 4

Worker A ficou esperando.

Mas antes do lock liberar:
novos workers chegam e pegam novas linhas livres.

Worker A continua esperando.

Isso pode repetir indefinidamente.

Ele não trava o sistema.
Ele simplesmente **nunca progride**.

Isso é starvation.

#### Diferença para deadlock

| Deadlock            | Starvation                |
| ------------------- | ------------------------- |
| ciclo de espera     | espera infinita sem ciclo |
| PostgreSQL detecta  | PostgreSQL não detecta    |
| erro retornado      | só lentidão               |
| rollback automático | não há rollback           |

#### Caso clássico em produção

Fila de pagamentos:

- workers rápidos pegam sempre os primeiros jobs
- um job específico fica sempre bloqueado
- sistema parece saudável
- mas certos pedidos nunca processam

Isso acontece sem erro no banco.

#### Onde aparece muito

- filas em banco
- controle de estoque
- `FOR UPDATE` concorrente
- sistemas com muitos retries agressivos

#### Solução principal

Use

```sql
FOR UPDATE SKIP LOCKED
```

Agora o worker não espera.
Ele pula linhas bloqueadas.

Resultado:

- ninguém fica preso
- todos progridem

#### Outra causa comúm

Transações longas + muitas curtas:

- curtas pegam lock rapidamente
- longa sempre perde a corrida
- nunca executa

Também é starvation

### Read replica

Fluxo típico em arquitetura:

```
POST /users → primary
GET /users → replica
```

Agora veja a sequência real.

---

#### Passo 1 — escrita

```sql
INSERT INTO users (email) VALUES ('ana@email.com');
COMMIT;
```

O PostgreSQL responde:

> sucesso

Porque o WAL foi persistido no **primário**

Mas a réplica ainda precisa:

1. receber WAL pela rede
2. aplicar replay
3. atualizar páginas

Isso leva milissegundos... às vezes segundos.

Isso é o **replication lag**.

---

#### Passo 2 — leitura imediata

Logo depois o frontend chama:

```sql
SELECT * FROM users WHERE email='ana@email.com';
```

Mas essa query vai para a réplica.

Resultado:

```
0 rows
```

Para a aplicação:

> o usuário não existe

Você criou um bug funcional.

---

#### Casos reais comuns

##### Login após cadastro

- usuário cria conta
- redireciona para login
- login falha

##### Checkout

- pedido criado
- página de confirmação consulta pedido
- pedido "não encontrado"

##### APIs REST

Sequência:

```
POST /orders
GET /orders/{id}
```

GET falha intermitentemente

Muito difícil de reproduzir localmente

#### Por que é traiçoeiro

- banco está saudável
- queries estão corretas
- não há erro SQL
- logs não mostram problema

É arquitetura.

---

#### Soluções

#### :one: Ready-your-writes (mais comum)

Após escrever, ler do primário por alguns segundos.

Exemplo:

- mesma request usa primary
- ou cookie de sessão direciona

---

#### :two: Sticky session

Usuário que escreveu continua lendo do primário temporariamente.

---

#### :three Checar lag

Alguns drivers verificam LSN aplicado antes de ler.

---

#### :four: Não usar réplica para dados críticos

- autenticação
- pagamentos
- criação imediata de recurso

Use réplica para:

- dashboards
- relatórios
- listagens

---

#### Conclusão

Read replica melhora escalabilidade, mas troca consistência imediata por desempenho.

Se a aplicação não considerar isso, surgem bugs funcionais.

#### Cache (Redis) x PostgreSQL

Muitos devs tentam resolver tudo com:

- índice
- `EXPLAIN ANALYZE`
- tuning

Mas às vezes o banco não está lento.
Ele está **sobrecarregado de leituras repetidas**.

Exemplo real:

Endpoint:

```
GET /product/10
```

Cada acesso executa:

```sql
SELECT * FROM products WHERE id = 10;
```

Se 20.000 usuários abrirem a página:

→ 20.000 queries idênticas.

O PostgreSQL consegue, mas:

- ocupa conexões
- consome CPU
- disputa cache
- atrapalha outras operações

Isso é problema de arquitetura, não de SQL.

#### Onde o cache entra

Você guarda o resultado:

```
key: product:10
value: JSON do produto
```

Fluxo:

1. aplicação consulta Redis
2. se existir → retorna
3. se não → consulta banco e popula cache

Isso chama:
**cache-aside pattern**

Agora:

20.000 acessos → 1 query real no banco.

#### Quando usar cache

Ideal para:

- catálogo de produtos
- perfis
- configurações
- rankings
- contadores de leitura
- páginas públicas

Não ideal para:

- saldo bancário
- estoque crítico
- autenticação imediata

#### O problema perigoso (muito comum)

Você atualiza o banco:

```sql
UPDATE products SET price = 100 WHERE id = 10;
```

Mas esquece de invalidar o cache.

O Redis ainda tem:

```
price = 120
```

Agora:

- banco está correto
- aplicação mostra dado antigo

Isso é:

> **cache incoerente (stale read)**

E é pior que banco lento, porque:

- não gera erro
- gera comportamento incorreto

#### Estratégia de invalidação

#### :one: TTL

Cache expira automaticamente (ex.: 60s).
Simples, porém inconsistente temporariamente.

#### :two: Invalidação ativa

Após UPDATE:

```
DEL product:10
```

Mais correto, mais complexo.

#### :three: Write-through

Escreve no banco e no cache juntos.

#### Regra importante

PostgreSQL → fonte da verdade

Cache → cópia descartável

Nunca confiar no cache para consistência.

### `SELECT COUNT(*) FROM tabela_grande`, qual o problema?

A pergunta real é:

> Por que o banco não guarda "quantidade de linhas" pronta?

Muitos bancos fazem isso.
O PostgreSQL não pode.

Por causa do **MVCC**

#### O problema

Lembre:

várias versões da mesma linha podem existir.

Situação

| id  | status             |
| --- | ------------------ |
| 1   | antiga (invisível) |
| 1   | nova (visível)     |

Uma transação antiga pode enxergar a antiga.
Outra pode enxergar a nova.

Então:

**o número de linhas depende da transação que pergunta**

Logo:

O banco não tem um único valor correto de contagem.

#### Por isso o COUNT precisa verificar

Para cada linha ele precicsa checar:

```
xmin / xmax → visivel neste snapshot?
```

Esse teste só existe no **heap**, não no índice.

Então mesmo com índice:

```sql
SELECT COUNT(*) FROM orders;
```

o PostgreSQL geralmente faz:

```
Seq Scan
```

E isso é O(n).

Tabela com 50 milhões de linhas → leitura massiva.

#### "Mas as vezes é rápido"

Pode ser rápido quando:

- tabela está no cache
- visibility map está completo
- index-only scan possível

Mas não é garantido

#### O que sistemas reais fazem

Eles não contam em tempo real.

#### Estratégia 1 — contador materializado

Tabela:

```
stats (orders_count)
```

Ao inserir pedido:

```sql
UPDATE stats SET orders_count = orders_count + 1;
```

---

#### Estratégia 2 — cache

Guardar no redis:

```
orders:count = 15420321
```

---

#### Estratégia 3 — estimativa

```sql
SELECT reltuples FROM pg_class WHERE relname='orders';
```

Rápido, mas aproximado.

Muito usado em paginação:

> "aprox. 15 milhões de registros"

---

#### Conclusão

`COUNT(*)` é lento não por falta de índice.

É lento porque o PostgreSQL precisa respeitar consistência transacional do MVCC.

### Nested Loop Join, Hash Join, Merge Join

Considere:

```sql
SELECT *
FROM orders o
JOIN users u ON u.id = o.user_id;
```

O PostgreSQL tem 3 maneiras principais de resolver isso.

#### :one: Nested Loop Join

Algoritmo:

```
para cada linha de orders
   procurar o user correspondente
```

Funciona assim internamente:

```
orders: 10 linhas
→ faz 10 buscas em users
```

Se `users.id` tem índice:
cada busca é rápida.

Quando é ótimo

- tabela externa pequena
- lookup por chave primária
- consultas por ID

Exemplo típico de API:

```sql
SELECT * FROM orders WHERE id = 10;
JOIN users ON ...
```

#### Quando vira desastre

Se `orders` tiver 1 milhão de linhas:

→ 1 milhão de buscas no índice.

Isso é o clássico:

**nested loop explosion**

#### :two: Hash Join

O banco faz:

1. lê `users`
2. cria tabela hash em RAM
3. percorre `orders`
4. consulta o hash

Ou seja:

```
users vira um mapa em memória
orders só consulta
```

#### Quando é ótimo

- tabelas grandes
- sem índice útil
- joins por igualdade

Exemplo:

```sql
JOIN users ON users.country = orders.country
```

#### Custo

- usa RAM (work_mem)
- pode fazer spill para disco se grande demais

#### :three: Merge Join

Funciona como dois ponteiros:

```
orders ordenado
users ordenado
anda simultaneamente
```

Nenhuma busca aleatória.

#### Quando é ótimo

- ambas ordenadas
- range join
- índices B-Tree compatíveis

Exemplo:

```sql
JOIN users ON users.id = orders.user_id
ORDER BY users.id
```

Muito eficiente para grandes conjuntos ordenados.

#### Por que isso importa

O planner decide baseado na estimativa de linhas.

Se estatística diz:

```
orders = 10 linhas
```

Ele escolhe nested loop.

Mas se na realidade:

```
orders = 5 milhões
```

Você ganha uma query extremamente lenta.

E a query parece “normal”.

#### Sintoma clássico em produção

Nada mudou no código.
Mas a tabela cresceu.

De repente:

- CPU do banco sobe
- endpoint específico fica lento

Você roda:

```sql
EXPLAIN ANALYZE
```

E vê:

```
Nested Loop (esperava 10 linhas)
real: 2.000.000
```

Isso quase sempre é:

> estatística incorreta → join errado

Regra prática

| Join        | Melhor cenário            |
| ----------- | ------------------------- |
| Nested Loop | poucos registros + índice |
| Hash Join   | muitos registros          |
| Merge Join  | dados ordenados           |

### Advisory lock

Normalmente você trava algo assim:

```sql
SELECT * FROM orders WHERE id = 10 FOR UPDATE;
```

Isso funciona quando:
→ o recurso é uma **linha existente**.

Mas e quando o recurso é lógico?

Exemplos:

- gerar fatura mensal do usuário
- processar webhook
- importar arquivo
- rodar cron em múltiplos servidores

Você não tem uma linha específica para travar.

Se duas instâncias da API rodarem:

→ você executa duas vezes.

#### O que o advisory lock faz

Você cria um “cadeado lógico”:

```sql
SELECT pg_advisory_lock(12345);
```

Agora, qualquer outra sessão que tentar o mesmo número:

```sql
SELECT pg_advisory_lock(12345);
```

vai esperar.

Você acabou de criar um **mutex distribuído usando o PostgreSQL**.

#### Exemplo real — webhook duplicado

Gateway de pagamento envia 2 vezes:

```
payment_id = 9981
```

Você:

```sql
SELECT pg_try_advisory_lock(9981);
```

Se retornar `false`:
→ outro worker já está processando.

Evita duplicidade.

#### Diferença principal

| FOR UPDATE         | Advisory lock        |
| ------------------ | -------------------- |
| trava linha física | trava recurso lógico |
| depende de tabela  | independente         |
| automático no MVCC | controlado pela app  |

#### Tipos

**Session lock**

```
pg_advisory_lock
```

Libera ao fechar conexão.

**Transaction lock**

```
pg_advisory_xact_lock
```

Libera no commit/rollback (mais seguro).

#### Quando usar

Use advisory lock quando:

- não existe linha para travar
- coordenação entre múltiplos serviços
- evitar processamento duplicado
- jobs distribuídos

#### Importante

Não substitui constraint.

Ele evita concorrência,
mas não garante integridade permanente.

Sempre combine com:

- unique index
- idempotência

### Replication Lag

é o tempo entre **commit no primary** e **aplicação na replica**.

Agora o ponto importante:

a réplica não é apenas “copiar dados”.

Ela faz:

1. receber WAL pela rede
2. gravar WAL localmente
3. aplicar mudanças página por página
4. atualizar índices

Isso chama **WAL replay**.

#### Por que o lag aparece

O primário produz WAL numa velocidade:

```
writes por segundo
```

A réplica consome numa velocidade:

```
replay por segundo
```

Se:

```
produção > consumo
```

→ o atraso cresce.

Isso ocorre quando:

- relatório pesado na réplica
- disco da réplica lento
- rede congestionada
- pico de escrita no primário
- autovacuum pesado

#### Efeito pouco conhecido (importante)

Lag não afeta só leitura.

Ele também afeta o **primário**.

Por quê?

O PostgreSQL não pode reciclar segmentos WAL que a réplica ainda não consumiu.

Resultado:

- diretório pg_wal cresce
- mais I/O
- checkpoints mais pesados

Ou seja:

> réplica lenta pode degradar o primário.

```sql
Como medir
SELECT application_name,
       write_lag,
       flush_lag,
       replay_lag
FROM pg_stat_replication;
```

Ou comparar:

```
pg_current_wal_lsn()
pg_last_wal_replay_lsn()
```

#### Sintoma clássico em aplicação

Usuário:

1. cria pedido
2. abre página de pedidos
3. não vê pedido
4. atualiza página → aparece

Intermitente.

Não é bug de código.
É lag.

#### Mitigação real

- rotas críticas → primário
- dashboards → réplica
- sticky session após escrita
- limitar carga analítica na réplica
- hardware similar ao primário

### Write amplification

Imagine uma tabela simples:

```
orders
(id, user_id, status, created_at, total)
```

Sem índices.

Você executa:

```sql
INSERT INTO orders VALUES (...);
```

O PostgreSQL faz basicamente:

1. escreve a linha no heap
2. grava no WAL
3. commit

Rápido.

Agora você “otimiza” leitura e cria:

```sql
CREATE INDEX idx_user       ON orders(user_id);
CREATE INDEX idx_status     ON orders(status);
CREATE INDEX idx_created    ON orders(created_at);
CREATE INDEX idx_total      ON orders(total);
CREATE INDEX idx_composite  ON orders(user_id, status);
```

Agora o MESMO INSERT faz:

1. escrever no heap
2. atualizar índice user_id
3. atualizar índice status
4. atualizar índice created_at
5. atualizar índice total
6. atualizar índice composto
7. escrever tudo no WAL
8. commit

Uma escrita virou **7 escritas físicas + WAL**.

Isso é write amplification.

#### E o UPDATE é pior

Lembra do MVCC:

`UPDATE` não altera linha → cria nova versão.

Então:

UPDATE orders SET status='paid' WHERE id=10;

O banco:

1. cria nova tupla no heap
2. invalida entradas antigas nos índices
3. cria novas entradas nos índices
4. gera WAL

Se houver 6 índices:
→ operação muito mais cara que parece.

#### Efeito colateral

Mais escrita → mais WAL:

- mais flush
- mais checkpoint
- mais I/O
- mais autovacuum

Resultado:

> leitura fica rápida <br>
> escrita vira gargalo do sistema

#### Sintoma clássico em produção

- sistema começa a crescer
- criam índices para melhorar SELECT
- semanas depois:
  - INSERT lento
  - fila de jobs aumenta
  - CPU do banco alta

Causa:

não é query pesada
é **escrita amplificada pelos índices**.

#### Regra prática

Índice só deve existir se:

> uma query crítica realmente o utiliza.

Índice “por garantia” é erro comum.

#### Como diagnosticar

Veja:

```sql
SELECT relname, idx_scan
FROM pg_stat_user_indexes;
```

Índices com idx_scan = 0:
→ nunca usados
→ só prejudicam escrita.

### Transação com muitas operações, `COMMIT` e picos de latência

Aqui existe um detalhe pouco intuitivo:

Durante a transação, o PostgreSQL **não garante durabilidade ainda**.

Ele vai fazendo:

```
UPDATE ...
INSERT ...
DELETE ...
```

Essas operações:

- geram WAL
- ficam em memória
- ainda não estão garantidas no disco

O banco só promete:

> “se eu cair agora, posso perder essa transação”.

A garantia só acontece no:

```sql
COMMIT;
```

#### O que realmente ocorre no COMMIT

No momento do commit o PostgreSQL precisa:

1. escrever todos os registros WAL pendentes
2. forçar gravação no disco (fsync)
3. marcar a transação como committed
4. liberar locks
5. tornar dados visíveis

Esse `fsync` é caro porque:
disco é milhares de vezes mais lento que RAM.

#### Por que muitas operações pioram

Imagine:

1000 inserts dentro de uma transação:

Durante execução:
→ tudo parece rápido

No final:

```sql
COMMIT;
```

Agora o banco precisa garantir a durabilidade de **todos os 1000**.

Então ocorre:

- grande flush de WAL
- espera de I/O
- outras transações aguardam

Sintoma:
a aplicação parece travar **no commit**, não na query.

#### Efeito colateral

Outras sessões também esperam.

Por quê?

O WAL flush é praticamente serializado:

- vários commits simultâneos
- todos esperam o mesmo disco

Isso cria:

> commit queue

Você vê:

- CPU baixa
- banco lento
- latência alta

Causa: I/O de durabilidade.

#### Caso clássico

Código:

```sql
BEGIN
for 10k registros:
   INSERT
COMMIT
```

Tudo rápido → último comando demora segundos.

Dev acha:

> “o commit é lento”

Na verdade:
o commit está pagando **todo o custo acumulado**.

#### Como resolver

- commits a cada N registros
- usar COPY em vez de INSERT massivo
- evitar transações gigantes
- fila de escrita

#### Conclusão

Queries rápidas não significam transação barata.

O custo real da escrita no PostgreSQL ocorre no:

> COMMIT (durabilidade via WAL).

### Alta concorrência

#### Código comum (errado)

```sql
BEGIN;

SELECT stock FROM inventory WHERE product_id = 10;

-- aplicação verifica se stock > 0

UPDATE inventory
SET stock = stock - 1
WHERE product_id = 10;

COMMIT;
```

Cenário com duas requisições simultâneas:

| Transação A | Transação B |
| ----------- | ----------- |
| SELECT → 1  |             |
|             | SELECT → 1  |
| UPDATE → 0  |             |
|             | UPDATE → -1 |

Resultado:
estoque negativo.

Mesmo com transação.

Por quê?

Porque `SELECT` não bloqueia atualização.

MVCC permite leitura concorrente.

### Solução correta 1 — UPDATE condicional atômico

```sql
UPDATE inventory
SET stock = stock - 1
WHERE product_id = 10
AND stock > 0;
```

Depois verifica:

```
rows_affected == 1 → sucesso
rows_affected == 0 → estoque insuficiente
```

Agora a validação e a escrita são **uma única operação atômica**.

Sem race condition.

### Solução correta 2 — SELECT FOR UPDATE

```sql
BEGIN;

SELECT stock
FROM inventory
WHERE product_id = 10
FOR UPDATE;

-- agora a linha está bloqueada

UPDATE inventory
SET stock = stock - 1
WHERE product_id = 10;

COMMIT;
```

Isso serializa as operações.

### Solução ainda mais robusta

Adicionar constraint:

```
ALTER TABLE inventory
ADD CONSTRAINT stock_non_negative
CHECK (stock >= 0);
```

Mesmo que lógica falhe, banco impede estado inválido.

### O que realmente estava acontecendo

Não era problema de isolamento baixo.

Era:

> validação feita fora da operação atômica.

Isso é padrão comum em APIs.
