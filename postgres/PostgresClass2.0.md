# PostgresClass 2.0 — PostgreSQL: fundamentos ao avançado

Conteúdo reorganizado para aprendizado incremental e fluxo linear, em ordem de complexidade crescente.

## Sumário

### Parte 1: Fundamentos

1. ACID
2. MVCC → Dead tuples → VACUUM → Wraparound
3. WAL
4. Checkpoints

### Parte 2: Isolamento e Concorrência

5. Níveis de isolamento
6. Tipos de locks (incluindo Advisory)
7. Deadlock
8. Contenção de locks e Hot Row Contention

### Parte 3: Planner e Busca

9. Estatísticas e ANALYZE
10. Sequential vs Index Scan
11. Índices (B-Tree, Hash, GIN, GiST, BRIN, Partial, Composite, Index Only Scan)
12. EXPLAIN ANALYZE e tipos de Join

### Parte 4: Operações

13. Paginação OFFSET vs Keyset
14. N+1, COUNT(\*), Write amplification

### Parte 5: Arquitetura

15. Pool de conexões e PgBouncer
16. Transações longas
17. Prepared Statements

### Parte 6: Alta Disponibilidade

18. Réplicas e Replication Lag
19. Failover
20. PITR
21. Replicação física vs lógica

### Parte 7: Práticas

22. Migrations
23. Idempotência
24. Fila e Starvation

### Parte 8: Diagnóstico

25. pg_stat_activity e pg_stat_statements

### Parte 9: Avançado

26. Cache (Redis) x PostgreSQL
27. Alta concorrência e race condition
28. Transação longa e pico no COMMIT

---

## Parte 1: Fundamentos

### 1. O que significa o acrônimo ACID?

#### A — Atomicidade

Exemplo clássico:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

Se cair entre as duas instruções:

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

E o user **não existir** → erro.

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

---

### 2. MVCC → Dead tuples → VACUUM → Transaction ID Wraparound

#### MVCC — o que realmente acontece dentro do PostgreSQL

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

#### Dead tuples — a consequência do MVCC

Quando ocorre:

```sql
UPDATE users SET name = 'Ana Maria' WHERE id = 1;
```

O PostgreSQL:

1. cria nova versão da linha
2. marca a antiga com `xmax = tx_id`

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

#### VACUUM — o que realmente faz

Lembra das dead tuples?
Elas continuam dentro do arquivo físico da tabela (`heap file`).

O VACUUM:

1. varre página por página do heap
2. verifica visibility (usando xmin/xmax)
3. confirma que nenhuma transação precisa mais daquela versão
4. marca o espaço como reutilizável (free space map)

Importante:

> Ele não apaga a linha do arquivo.
> Ele só permite que uma próxima inserção utilize aquele espaço.

Por isso o arquivo não diminui.

#### Visibility Map (conceito importante)

Durante o VACUUM o PostgreSQL marca páginas como:

- **all-visible**
- **all-frozen**

Isso permite:

- index-only scan (veremos na seção de índices)
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

---

### 3. WAL (Write-Ahead Log)

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

Quando você dá `COMMIT`, o PostgreSQL **não precisa salvar a tabela ainda**.

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

---

### 4. Checkpoints

Quando você executa:

```sql
UPDATE orders SET status = 'paid' WHERE id = 10;
```

O PostgreSQL:

1. escreve no WAL
2. altera a página na RAM (shared_buffers)

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

---

## Parte 2: Isolamento e Concorrência

### 5. Níveis de isolamento no PostgreSQL

#### Os níveis no PostgreSQL

O padrão SQL define:

1. READ UNCOMMITTED
2. READ COMMITTED
3. REPEATABLE READ
4. SERIALIZABLE

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

**Phantom Read (muito perguntado)**
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
`SERIALIZIBLE` ou locking explícito.

---

### 6. Tipos de locks (incluindo Advisory Locks)

#### Row-Level Locks (mais comuns)

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

#### Table-Level Locks

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

#### Advisory Locks

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

**O que o advisory lock faz**

Você cria um "cadeado lógico":

```sql
SELECT pg_advisory_lock(12345);
```

Agora, qualquer outra sessão que tentar o mesmo número:

```sql
SELECT pg_advisory_lock(12345);
```

vai esperar.

Você acabou de criar um **mutex distribuído usando o PostgreSQL**.

**Exemplo real — webhook duplicado**

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

**Diferença principal**

| FOR UPDATE         | Advisory lock        |
| ------------------ | -------------------- |
| trava linha física | trava recurso lógico |
| depende de tabela  | independente         |
| automático no MVCC | controlado pela app  |

**Tipos**

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

**Quando usar**

Use advisory lock quando:

- não existe linha para travar
- coordenação entre múltiplos serviços
- evitar processamento duplicado
- jobs distribuídos

Muito usado com Node.js + múltiplas instâncias.

**Importante**

Não substitui constraint.
Sempre combine com:
<<<<<<< HEAD
=======

> > > > > > > bac2437 (adds organized version of `PostgresClass`)

- unique index
- idempotência

#### Impacto em backend (caso real)

Em uma API Node.js com pool de conexões:

- muitas requisições concorrentes
- cada uma abre transação
- atualiza mesma tabela

Se a ordem de acesso for inconsistente → deadlock.
Se transações forem longas → lock retention alto.
Se houver DDL em horário errado → freeze da aplicação.

---

### 7. Deadlock

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

---

### 8. Contenção de locks e Hot Row Contention

#### Contenção de locks

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

1. **WAL flush** — Commit precisa garantir WAL persistido no disco. Se 500 commits simultâneos → todos esperam o disco.

2. **Index page contention** — Se muitos inserts usam o mesmo índice crescente (inserções no final da B-Tree), todos disputam a mesma página do índice.

3. **LWLocks (locks internos)** — PostgreSQL tem locks internos para: buffer cache, freelist, WAL buffers, clog. Esses não aparecem como row locks, mas geram espera.

4. **Context switching** — 500 conexões = 500 processos. Mesmo com CPU sobrando: scheduler troca contexto, cache de CPU invalida, overhead cresce. Throughput começa a cair.

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

#### Solução arquitetural

- limitar pool
- usar PgBouncer
- batch writes
- reduzir commits desnecessários
- evitar hot rows
- shardear se necessário

#### Hot Row Contention

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

**O que acontece dentro do PostgreSQL**

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

**Como identificar**

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

**Por que MVCC não resolve**

MVCC permite:

SELECT não bloquear UPDATE.

Mas UPDATE x UPDATE na mesma linha
→ precisa serializar.

Não tem como duas versões modificarem o mesmo registro simultaneamente.

**Soluções reais**

**Errado (modelo ingênuo):** contador central

```
likes = likes + 1
saldo = saldo - 1
estoque = estoque - 1
```

**Solução 1 — Tabela de eventos**

Ao invés de atualizar:

```sql
INSERT INTO stock_movements(product_id, delta) VALUES (10, -1);
```

Depois agrega:

```sql
SELECT SUM(delta) FROM stock_movements WHERE product_id = 10;
```

Agora: cada compra escreve em linha diferente → paralelizável.

**Solução 2 — Sharding lógico**

Em vez de 1 contador:

```
counter_1
counter_2
counter_3
counter_4
```

Cada request usa um shard aleatório. Depois soma.

**Solução 3 — Fila**

- grava pedido
- worker sequencial desconta estoque

Remove concorrência.

**Solução 4 — Cache**

Redis INCR/DECR para contadores de alta frequência.

**Conclusão importante**

Hot row contention é um problema **de modelagem de dados**, não de índice nem hardware

Você não resolve aumentando CPU.
Você resolve **mudando o padrão de escrita**.

**Conclusão geral**

PostgreSQL escala melhor com:

> poucas conexões bem utilizadas
> do que muitas conexões concorrendo

Concorrência excessiva reduz eficiência.

---

## Parte 3: Planner e Busca

### 9. Estatísticas e ANALYZE

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

#### Onde entram as estatísticas

O PostgreSQL mantém um resumo matemático das tabelas no catálogo interno: `pg_statistic`

Quem cria esse resumo é o `ANALYZE`.

O `ANALYZE` **não lê a tabela inteira**.
Ele faz amostragem e tenta responder:

> "Se alguém filtrar esta coluna, quantas linhas provavelmente vão aparecer?"

Ele coleta:

1. **MCV (Most Common Values)** — Valores mais frequentes.

| valor     | frequencia |
| --------- | ---------- |
| paid      | 92%        |
| pending   | 7%         |
| cancelled | 1%         |

2. **Histogram** — Distribuição dos demais valores (usado para ranges). Exemplo: `created_at > '2025-01-01'`

**:four: null_frac**

4. **null_frac** — Percentual de NULLs. Afeta seletividade de filtros.

#### O que o planner realmente quer saber

Ele não quer saber **o valor**.
Ele quer saber quantas linhas a query vai retornar.

Isso se chama: **cardinality estimation**.

E isso decide o plano.

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

---

### 10. Como o PostgreSQL busca os dados — Sequential vs Index Scan

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

1. consulta a B-tree
2. encontra ponteiro da linha
3. vai até o heap buscar a tupla

Problema: isso gera acesso aleatório ao disco.

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
- colunas booleanas

Exemplos bons:

- email
- CPF
- id
- uuid

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

---

### 11. Índices do PostgreSQL

#### 1. B-Tree (90% dos casos)

```sql
CREATE INDEX idx_users_email ON users(email);
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

#### 2. Hash

```sql
CREATE INDEX idx_users_email_hash ON users USING HASH(email)
```

Só acelera:

```
WHERE email = ?
```

Não serve para range nem ordenação.
Na prática quase nunca compensa usar em vez de B-Tree.

#### 3. GIN (muito importante em APIs modernas)

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

#### 4. GiST

Usado quando você não quer igualdade, mas **proximidade**.

Exemplo:

- distância geográfica
- ranges de data
- similaridade

Comum em PostGIS
"restaurantes perto de mim".

#### 5. BRIN (muito subestimado)

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

#### O comportamento normal de um índice

Mesmo quando um índice é usado:

```sql
SELECT email FROM users WHERE email = 'ana@email.com';
```

O PostgreSQL:

1. consulta a B-Tree
2. encontra o ponteiro (TID)
3. vai até a tabela (heap) buscar a linha

Ou seja:

> Índice não guarda a linha, só o endereço dela.

Isso se chama:
**heap fetch**

E isso custa caro.

#### Index Only Scan (depende do VACUUM)

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

**O problema da visibilidade**

O índice não sabe se a linha ainda é visível para sua transação.
(lembrando: múltiplas versões da mesma linha existem)

Então o PostgreSQL precisa confirmar:

> essa tupla não foi deletada ou atualizada depois?

Quem fornece essa informação é o **visibility map** (atualizado pelo VACUUM).

Ele marca páginas do heap como:

- todas as linhas visíveis
- seguras para leitura direta

**Quem atualiza o visibility map?**

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

Benefícios:

- índice pequeno
- menos WAL
- menos checkpoint
- mais cache

Muito usado em:

- filas
- soft delete (`deleted_at IS NULL`)
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

**Regra prática (muito perguntada)**

Coluna mais seletiva primeiro?
**Não exatamente.**

A regra real:

> coloque primeiro a coluna mais usada como filtro inicial das queries

#### Impacto em produção

Índice também tem custo:

Cada `INSERT/UPDATE`:

- atualiza todos os índices
- gera WAL
- aumenta checkpoint

Índice demais → write performance piora.

---

### 12. EXPLAIN ANALYZE e tipos de Join

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

Isso significa:

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

#### Como você diagnostica isso

Você usa `EXPLAIN ANALYZE`.

Ele compara:

| estimado | real      |
| -------- | --------- |
| rows=5   | rows=1200 |

Diferença grande → problema de estatística.

Solução:

```sql
ANALYZE orders;
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

#### Nested Loop Join, Hash Join, Merge Join

Considere:

```sql
SELECT *
FROM orders o
JOIN users u ON u.id = o.user_id;
```

O PostgreSQL tem 3 maneiras principais de resolver isso.

#### 1. Nested Loop Join

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

**Quando vira desastre**

Se `orders` tiver 1 milhão de linhas:

→ 1 milhão de buscas no índice.

Isso é o clássico:

**nested loop explosion**

#### 2. Hash Join

O banco faz:

1. lê `users`
2. cria tabela hash em RAM
3. percorre `orders`
4. consulta o hash

Quando é ótimo

- tabelas grandes
- sem índice útil
- joins por igualdade

Custo:

- usa RAM (work_mem)
- pode fazer spill para disco se grande demais

#### 3. Merge Join

Funciona como dois ponteiros:

```
orders ordenado
users ordenado
anda simultaneamente
```

Nenhuma busca aleatória.

Quando é ótimo

- ambas ordenadas
- range join
- índices B-Tree compatíveis

**Regra prática**

| Join        | Melhor cenário            |
| ----------- | ------------------------- |
| Nested Loop | poucos registros + índice |
| Hash Join   | muitos registros          |
| Merge Join  | dados ordenados           |

---

## Parte 4: Operações

### 13. Paginação: OFFSET x Keyset

#### OFFSET pagination

Exemplo clássico:

```sql
SELECT * FROM orders
ORDER BY id
LIMIT 20 OFFSET 100000;
```

O que o PostgreSQL faz:

Ele **não pula** 100 mil linhas.

Ele:

1. percorre 100000 linhas
2. descarta
3. só então retorna 20

Ou seja:

- lê índice
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
- continua a partir do ponto
- não precisa percorrer o passado

Complexidade:

**O(log n)** ao invés de **O(n)**.

#### Por que escala muito melhor

O índice B-Tree já é ordenado.

O banco só faz:

> continuar a navegação da árvore

Nenhum descarte de linhas.

---

### 14. N+1 queries, COUNT(\*) e Write amplification

#### O problema de N+1 queries

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

**Por que ORMs causam**

Exemplo típico (pseudo-ORM)

```js
const users = await User.findAll();

for (const user of users) {
  await user.getOrders();
}
```

Isso ativa lazy loading.

**Forma correta**

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

#### SELECT COUNT(\*) FROM tabela_grande

A pergunta real é:

> Por que o banco não guarda "quantidade de linhas" pronta?

Muitos bancos fazem isso.
O PostgreSQL não pode.

Por causa do **MVCC**

Várias versões da mesma linha podem existir.
O número de linhas depende da transação que pergunta.
O banco não tem um único valor correto de contagem.

Para cada linha o PostgreSQL precisa checar:

```
xmin / xmax → visível neste snapshot?
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

**O que sistemas reais fazem**

- contador materializado
- cache (Redis)
- estimativa: `SELECT reltuples FROM pg_class WHERE relname='orders';`

#### Write amplification

Imagine uma tabela simples sem índices.

Você executa:

```sql
INSERT INTO orders VALUES (...);
```

O PostgreSQL faz basicamente:

1. escreve a linha no heap
2. grava no WAL
3. commit

Rápido.

Agora você cria vários índices:

```sql
CREATE INDEX idx_user       ON orders(user_id);
CREATE INDEX idx_status     ON orders(status);
CREATE INDEX idx_created    ON orders(created_at);
...
```

Agora o MESMO INSERT faz:

1. escrever no heap
2. atualizar cada índice
3. escrever tudo no WAL
4. commit

Uma escrita virou **múltiplas escritas físicas + WAL**.

E o UPDATE é pior: MVCC cria nova versão + atualiza todos os índices.

**Regra prática**

Índice só deve existir se:

> uma query crítica realmente o utiliza.

**Como diagnosticar**

```sql
SELECT relname, idx_scan
FROM pg_stat_user_indexes;
```

Índices com idx_scan = 0:
→ nunca usados
→ só prejudicam escrita.

---

## Parte 5: Arquitetura

### 15. Pool de conexões e PgBouncer

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

Você limita a concorrência no ponto correto: **antes do banco**.

#### PgBouncer (muito perguntado)

**O problema real que existe sem PgBouncer**

Você tem:

- API Node.js
- 4 pods
- cada pod com pool = 20 conexões

Total: 4 x 20 = 80 conexões no PostgreSQL

Agora acontece pico de tráfego e autoscaling → **20 pods**.

Agora: 20 x 20 = 400 conexões abertas no banco

No PostgreSQL isso é grave porque:

> conexão não é só um socket — é um processo inteiro do servidor.

**O que o PgBouncer faz**

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

Ou seja: 400 sessões lógicas → 30 sessões físicas

**Por que isso é especialmente importante no PostgreSQL**

O PostgreSQL:

- não foi projetado para milhares de conexões
- foi projetado para **poucas conexões ativas bem utilizadas**

| Conexões | Efeito                |
| -------- | --------------------- |
| 20-50    | ótimo                 |
| 100-200  | aceitável             |
| 400+     | degradação séria      |
| 1000+    | quase sempre instável |

Cada backend mantém snapshot do MVCC.
Com muitas conexões abertas (mesmo ociosas), o banco assume que alguma pode precisar de dados antigos.
Resultado: VACUUM não consegue limpar.

**Diferença entre pool de aplicação e PgBouncer**

- **Pool do Node**: controla concorrência dentro de um único processo. Cada instância cria o próprio pool → o total explode.
- **PgBouncer**: controla concorrência **global do banco**. O "porteiro" que limita quantas conexões físicas o banco aceita.

---

### 16. O problema de transações longas

Lembre do MVCC:

cada transação enxerga o banco como ele estava **no momento do BEGIN**.

Transação A fica aberta (ex.: usuário abriu página e deixou o navegador parado):

```sql
BEGIN;
SELECT * FROM orders; -- snapshot tirado agora
```

Enquanto isso o sistema continua:

```sql
UPDATE orders ...
DELETE orders ...
INSERT orders ...
```

O banco vai criando novas versões e versões antigas virando dead tuples.

Normalmente o VACUUM limparia.

Mas:

> O PostgreSQL não pode apagar uma linha antiga se ainda existir uma transação que possa enxergá-la.

O VACUUM fica impedido.

#### O efeito dominó

Quanto mais tempo a transação fica aberta:

1. mais versões antigas acumulam
2. tabela cresce
3. índice cresce
4. cache piora
5. queries ficam mais lentas
6. checkpoint mais pesado
7. WAL cresce

Você observa: "o banco foi ficando lento ao longo do dia"

#### O caso clássico em Node.js

```js
BEGIN
SELECT ...
await chamarAPIexterna() // 3 segundos
UPDATE ...
COMMIT
```

Durante esses 3 segundos o banco inteiro fica impedido de limpar versões antigas.

#### O pior cenário

Se durar horas: o banco se aproxima do **transaction id wraparound**.

O PostgreSQL entra em modo de proteção e começa a bloquear writes.

Uma única conexão esquecida pode degradar todo o banco.

---

### 17. Prepared Statements

Quando você executa normalmente:

```sql
SELECT * FROM orders WHERE user_id = 10;
```

O PostgreSQL faz 3 etapas toda vez:

1. **parse** - entende o SQL
2. **plan** - decide índice/scan/join
3. **execute**

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
```

Agora o banco não reinterpreta SQL nem recalcula plano toda vez.

Drivers como `pg` do Node já fazem isso implicitamente.

#### O problema com pool / PgBouncer

Prepared statement vive **dentro da conexão**.

Com PgBouncer em _transaction pooling_:

request A usa conexão 3 → prepara statement
request B vai para conexão 7 → statement não existe

Erro clássico:

```
prepared statement "S_1" does not exist
```

Ou pior: **parameter sniffing** — plano cacheado para um valor raro usado no primeiro request, reutilizado para todos.

**Soluções comuns**

- desabilitar prepare no driver
- usar PgBouncer session mode
- `prepareThreshold` tuning

---

## Parte 6: Alta Disponibilidade e Backup

### 18. Réplicas e Replication Lag

Lembra do WAL?

Toda alteração no PostgreSQL primeiro vai para o WAL.

O primário faz:

```
UPDATE → grava WAL → commit
```

Na replicação: o primário também **envia o WAL via rede** para outro servidor.

O standby recebe:

```
recebe WAL → reexecuta alterações → atualiza seus data files
```

Ele não executa SQL da aplicação. Ele apenas **reaplica as mudanças**.

Isso se chama: **physical streaming replication**

#### O que é uma read replica

- SELECT permitido
- UPDATE/INSERT/DELETE proibido

Arquitetura:

```
API
 ├── writes → primary
 └── reads → replica
```

#### Replication Lag — o problema prático

O lag é o tempo entre **commit no primary** e **aplicação na replica**.

A réplica faz:

1. receber WAL pela rede
2. gravar WAL localmente
3. aplicar mudanças página por página
4. atualizar índices

Isso chama **WAL replay**.

**Por que o lag aparece**

```
produção de WAL (primário) > consumo (réplica)
```

→ o atraso cresce.

Quando: relatório pesado na réplica, disco lento, rede congestionada, pico de escrita.

**Efeito pouco conhecido**

Lag não afeta só leitura. Ele também afeta o **primário**.

O PostgreSQL não pode reciclar segmentos WAL que a réplica ainda não consumiu.

Resultado: diretório pg_wal cresce, mais I/O, checkpoints mais pesados.

> réplica lenta pode degradar o primário.

**Sintoma clássico em aplicação**

Fluxo: POST /users → primary, GET /users → replica

1. usuário cria conta
2. redireciona para login
3. login falha (dados ainda não na réplica)

Ou: pedido criado, página de confirmação não encontra o pedido.

**Soluções**

- **read-your-writes**: após escrever, ler do primário por alguns segundos
- **sticky session**: usuário que escreveu continua lendo do primário temporariamente
- não usar réplica para: autenticação, pagamentos, criação imediata de recurso
- usar réplica para: dashboards, relatórios, listagens

#### Como medir

```sql
SELECT application_name, write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
```

---

### 19. Failover

Failover significa: **promover standby a primary** quando o primário morre.

#### Risco de perda de dados

**Replicação assíncrona (default)**

Fluxo: commit no primary → responde cliente → envia WAL para replica

Se o primary cair entre commit e envio do WAL → standby nunca recebeu aquela transação.

**window of data loss**

**Replicação síncrona**

commit → envia WAL → replica confirma → primary responde cliente

Não há perda, mas: commit mais lento, depende da latência da rede.

#### Conclusão arquitetural

| Sistema          | Estratégia |
| ---------------- | ---------- |
| financeiro       | síncrona   |
| e-commerce comum | assíncrona |
| analytics        | assíncrona |

---

### 20. PITR (Point-in-Time Recovery)

O WAL é um **registro cronológico de tudo que aconteceu no banco**.

#### O problema do backup comum

Backup às 02:00. Às 15 alguém executa `DELETE FROM users;`.

Restaurar o backup = perder 13 horas de dados legítimos.

#### O que o PITR faz

Você arquiva os WALs continuamente.

Agora o PostgreSQL pode:

1. restaurar o backup das 02:00
2. reexecutar (replay) todos os WALs
3. **parar exatamente às 14:59:59**

Viagem no tempo do banco.

#### Por que isso é crítico em produção

- DELETE sem WHERE
- bug em código
- migration errada
- ransomware

---

### 21. Replicação física vs lógica

**Replicação física (streaming replication)**

O primário envia WAL → standby reaplica.

- estrutura idêntica
- índices idênticos
- não pode alterar schema
- serve para: failover, leitura

**Replicação lógica**

O PostgreSQL decodifica o WAL e transforma page changes em:

```sql
INSERT INTO orders ...
UPDATE users ...
```

Agora você pode:

- escolher tabelas
- enviar para outro PostgreSQL
- enviar para Kafka
- alimentar data warehouse (CDC — Change Data Capture)

**Trade-offs**

| Física      | Lógica      |
| ----------- | ----------- |
| rápida      | mais pesada |
| simples     | complexa    |
| consistente | flexível    |
| tudo        | seletiva    |
| HA          | integração  |

---

## Parte 7: Práticas

### 22. Migrations

#### O que derruba a aplicação

```sql
ALTER TABLE users ADD COLUMN age INT;
```

O PostgreSQL pega **ACCESS EXCLUSIVE LOCK**.

- ninguém lê, escreve ou atualiza
- pool esgotado
- requisições acumulando
- timeout

**Forma correta para índice**

```sql
CREATE INDEX CONCURRENTLY idx_orders_user ON orders(user_id);
```

Tabela continua operando; banco cria índice em background.

**Mudança de coluna (o mais perigoso)**

```sql
ALTER TABLE orders ALTER COLUMN price TYPE numeric(12,2)
```

Força **table rewrite** → pode travar produção por horas.

**Estratégia de zero-downtime**

1. adiciona nova coluna nullable
2. código passa a escrever nas duas
3. backfill em batches
4. muda leitura
5. remove coluna antiga

---

### 23. Idempotência no PostgreSQL

**Estratégia 1 — Idempotency Key**

```sql
CREATE UNIQUE INDEX idx_idempotency ON payments(idempotency_key)
```

```sql
INSERT INTO payments (...)
VALUES (...)
ON CONFLICT (idempotency_key)
DO NOTHING;
```

**Estratégia 2 — Unique constraint lógica**

```sql
CREATE UNIQUE INDEX idx_unique_order ON payments(order_id)
```

**Estratégia 3 — Lock explícito**

```sql
SELECT ... FOR UPDATE;
```

Ou advisory lock.

Idempotência **não é lógica de aplicação**. É restrição garantida pelo banco.

---

### 24. Fila usando PostgreSQL e Starvation

Tabela:

```
jobs (id, payload, status, created_at)
```

#### O erro comum

```sql
SELECT * FROM jobs WHERE status='pending' LIMIT 1;
```

Dois workers podem pegar o mesmo job. Race condition.

#### A solução: FOR UPDATE SKIP LOCKED

```sql
BEGIN;

SELECT id, payload
FROM jobs
WHERE status='pending'
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 1;

UPDATE jobs SET status='processing' WHERE id = ?;

COMMIT;
```

- `FOR UPDATE`: bloqueia linha
- `SKIP LOCKED`: evita fila de espera — outros workers pulam e pegam outro job

#### Starvation — o que pode acontecer

Sem `SKIP LOCKED`:

```sql
SELECT * FROM jobs WHERE status='pending' LIMIT 1 FOR UPDATE;
```

Cenário: Worker A tenta lockar linha 1. Worker B pega linha 2, C pega 3, D pega 4...

Worker A fica esperando. Novos workers chegam e pegam novas linhas livres.
Worker A continua esperando. Pode repetir indefinidamente.

**Solução**: usar `FOR UPDATE SKIP LOCKED` — ninguém fica preso.

#### Benefícios e limitações da fila em PostgreSQL

- transacional, sem duplicação, sem serviço externo, respeita ACID
- excelente para: emails, webhooks, retry jobs, processamento interno
- não substitui Kafka/RabbitMQ quando: milhões de jobs/minuto, processamento distribuído pesado

| Deadlock            | Starvation                |
| ------------------- | ------------------------- |
| ciclo de espera     | espera infinita sem ciclo |
| PostgreSQL detecta  | PostgreSQL não detecta    |
| rollback automático | não há rollback           |

---

## Parte 8: Diagnóstico

### 25. pg_stat_activity e pg_stat_statements

Quando alguém diz "o banco está lento", existem duas perguntas diferentes:

1. O que está acontecendo **agora**?
2. Qual query está custando mais ao longo do tempo?

**pg_stat_activity (tempo real)**

```sql
SELECT pid, state, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active';
```

Revela: queries rodando há minutos, transações abertas, bloqueios, `idle in transaction` (perigoso — impede VACUUM).

**pg_stat_statements (histórico agregado)**

Precisa estar habilitado (`shared_preload_libraries`).

```sql
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

**Como usar em incidente**

1. Ver bloqueio: `WHERE wait_event IS NOT NULL`
2. Ver long transactions: `now() - xact_start`
3. Ver queries mais custosas: `ORDER BY total_exec_time DESC`

| Ferramenta         | Serve para                   |
| ------------------ | ---------------------------- |
| pg_stat_activity   | diagnóstico imediato         |
| pg_stat_statements | tuning e otimização continua |

---

## Parte 9: Avançado

### 26. Cache (Redis) x PostgreSQL

Muitos devs tentam resolver tudo com índice e tuning.

Mas às vezes o banco está **sobrecarregado de leituras repetidas**.

20.000 usuários abrindo GET /product/10 → 20.000 queries idênticas.

**Cache-aside pattern**

1. aplicação consulta Redis
2. se existir → retorna
3. se não → consulta banco e popula cache

**Problema perigoso (muito comum)**

Você atualiza o banco mas esquece de invalidar o cache.

→ **cache incoerente (stale read)** — pior que banco lento.

**Regra**: PostgreSQL → fonte da verdade. Cache → cópia descartável.

---

### 27. Alta concorrência — race condition e soluções

**Código comum (errado)**

```sql
BEGIN;
SELECT stock FROM inventory WHERE product_id = 10;
-- aplicação verifica se stock > 0
UPDATE inventory SET stock = stock - 1 WHERE product_id = 10;
COMMIT;
```

Duas requisições simultâneas: ambas veem stock=1, ambas fazem UPDATE → estoque negativo.

Por quê? SELECT não bloqueia. MVCC permite leitura concorrente.

**Solução 1 — UPDATE condicional atômico**

```sql
UPDATE inventory
SET stock = stock - 1
WHERE product_id = 10 AND stock > 0;
```

rows_affected == 1 → sucesso; rows_affected == 0 → estoque insuficiente.

**Solução 2 — SELECT FOR UPDATE**

```sql
BEGIN;
SELECT stock FROM inventory WHERE product_id = 10 FOR UPDATE;
UPDATE inventory SET stock = stock - 1 WHERE product_id = 10;
COMMIT;
```

**Solução 3 — constraint**

```sql
ALTER TABLE inventory ADD CONSTRAINT stock_non_negative CHECK (stock >= 0);
```

O problema não era isolamento. Era: **validação feita fora da operação atômica**.

---

### 28. Transação com muitas operações e pico no COMMIT

Durante a transação, o PostgreSQL **não garante durabilidade ainda**.

Ele só promete durabilidade no `COMMIT`.

**O que ocorre no COMMIT**

1. escrever todos os registros WAL pendentes
2. forçar gravação no disco (fsync)
3. marcar transação como committed
4. liberar locks
5. tornar dados visíveis

O `fsync` é caro. Disco é milhares de vezes mais lento que RAM.

**Por que muitas operações pioram**

1000 INSERTs dentro de uma transação: durante execução tudo parece rápido.

No final, `COMMIT` precisa garantir durabilidade de **todos os 1000** → grande flush de WAL, espera de I/O.

O WAL flush é praticamente serializado. Vários commits simultâneos → todos esperam o mesmo disco → **commit queue**.

**Como resolver**

- commits a cada N registros
- usar COPY em vez de INSERT massivo
- evitar transações gigantes
- fila de escrita

**Conclusão**

O custo real da escrita no PostgreSQL ocorre no: **COMMIT (durabilidade via WAL)**.
