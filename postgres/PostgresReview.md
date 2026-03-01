# Postgres

## Índice

- [1) O que é MVCC e qual problema resolve?](#1-o-que-e-mvcc-multi-version-concurrency-control-no-postgresql-e-qual-problema-de-concorrencia-ele-resolve-na-pratica-dentro-de-um-sistema-com-multiplas-transacoes-simultaneas)
- [2) O que é dead tuple e consequência do MVCC?](#2-o-que-e-uma-dead-tuple-no-postgresql-e-por-que-ela-e-uma-consequencia-inevitavel-do-mvcc)
- [3) Como o VACUUM funciona e diferença VACUUM/VACUUM FULL/autovacuum?](#3-explique-como-o-vacuum-funciona-internamente-e-qual-a-diferenca-entre)
- [4) O que é o WAL e por que escreve primeiro nele?](#4-o-que-e-o-wal-write-ahead-log-no-postgresql-e-por-que-o-banco-escreve-primeiro-no-wal-antes-de-gravar-nas-tabelas)
- [5) O que é checkpoint e relação com WAL?](#5-explique-o-que-e-um-checkpoint-no-postgresql-e-como-ele-se-relaciona-diretamente-com-o-wal-e-com-picos-de-latencia-em-producao)
- [6) Propriedades ACID e WAL/MVCC?](#6-explique-as-propriedades-acid-de-uma-transacao-e-principalmente-qual-delas-o-wal-garante-diretamente-e-qual-o-mvcc-ajuda-a-implementar)
- [7) Níveis de isolamento no PostgreSQL](#7-explique-os-niveis-de-isolamento-no-postgresql-e-diga)
- [8) O que é deadlock e o que o banco faz?](#8-o-que-e-um-deadlock-no-postgresql-como-ele-acontece-entre-duas-transacoes-e-o-que-o-banco-faz-quando-detecta-um)
- [9) Tipos de locks (row-level, table-level, advisory)](#9-quais-sao-os-tipos-de-locks-no-postgresql-row-level-table-level-e-advisory-e-quando-cada-um-e-usado-na-pratica)
- [10) Diferença entre Sequential Scan, Index Scan e Index Only Scan](#10-qual-a-diferenca-entre)
- [11) O que é EXPLAIN ANALYZE](#11-o-que-e-um-explain-analyze-e-qual-a-diferenca-entre)
- [12) Tipos de índices no PostgreSQL](#12-quais-sao-os-tipos-principais-de-indices-no-postgresql-e-em-que-tipo-de-dado-ou-consulta-cada-um-deve-ser-usado)
- [13) Covering index (index-only scan) e visibility map](#13-o-que-e-um-covering-index-index-only-scan-e-qual-relacao-ele-tem-com-o-visibility-map-e-o-vacuum)
- [14) Partial index e composite index](#14-explique-o-que-e-um-partial-index-e-um-composite-index-e-em-qual-situacao-um-indice-composto-na-ordem-errada-deixa-de-ser-utilizado-pelo-planner)
- [15) OFFSET vs cursor (keyset) pagination](#15-explique-a-diferenca-entre-offset-pagination-e-curosr-keyset-pagination-e-por-que-offset-degrada-drasticamente-performance-em-tabelas-grandes)
- [16) Connection pooling e Node.js](#16-o-que-e-connection-pooling-no-postgresql-e-por-que-uma-aplicacao-nodejs-pode-derrubar-um-banco-apenas-abrindo-muitas-conexoes-simultaneas)
- [17) Transações longas e VACUUM/MVCC](#17-em-uma-api-concorrente-por-que-transacoes-longas-sao-perigosas-no-postgresql-e-como-elas-afetam-diretamente-o-vacuum-mvcc-e-crescimento-da-tabela)
- [18) Replicação e read replica](#18-o-que-e-uma-replicacao-streaming-replication-no-postgresql-o-que-e-uma-read-replica-e-qual-problema-ela-resolve-e-qual-problema-ela-nao-resolve)
- [19) Failover e risco de perda de dados](#19-o-que-e-failover-no-postgresql-e-qual-e-o-risco-de-perda-de-dados-dependendo-do-tipo-de-replicacao-configurada)
- [20) Migrations zero downtime](#20-o-que-sao-migrations-com-zero-downtime-no-postgresql-e-por-que-um-simples-alter-table-pode-derrubar-uma-aplicacao-em-producao)
- [21) Prepared statements e pool](#21-o-que-sao-prepared-statements-no-postgresql-e-por-que-eles-podem-melhorar-performance-e-tambem-causar-problemas-quando-usados-incorretamente-com-pool-de-conexoes)
- [22) Problema N+1 queries](#22-o-que-e-o-problema-n1-queries-por-que-orms-frequentemente-causam-isso-e-qual-o-impacto-real-no-postgresql-sob-carga-concorrente)
- [23) Idempotência em API](#23-como-voce-implementaria-idempotencia-em-uma-api-usando-postgresql-para-evitar-por-exemplo-dupla-cobranca-de-pagamento)
- [24) Fila (queue) usando PostgreSQL](#24-como-voce-implementaria-uma-fila-queue-usando-postgresql-garantindo-que-multiplos-workers-processem-jobs-sem-pegar-o-mesmo-item-duas-vezes)
- [25) ANALYZE e estatísticas](#25-o-que-e-analyze-no-postgresql-que-tipo-de-estatistica-ele-coleta-e-por-que-estatisticas-desatualizadas-fazem-o-planner-escolher-planos-ruins)
- [26) pg_stat_activity e pg_stat_statements](#26-o-que-e-o-pg_stat_activity-e-pg_stat_statements-e-como-voce-usaria-ambos-para-investigar-uma-producao-lenta)
- [27) PITR (Point-in-Time Recovery)](#27-o-que-e-pitr-point-in-time-recovery-no-postgresql-e-como-ele-utiliza-o-wal-para-restaurar-o-banco-ate-um-ponto-especifico-no-tempo)
- [28) Replicação física vs lógica](#28-qual-a-diferenca-entre-replicacao-fisica-e-replicacao-logica-no-postgresql-e-quando-voce-escolheria-cada-uma)
- [29) Contenção de locks sem deadlocks](#29-por-que-muitas-conexoes-simultaneas-executando-transacoes-curtas-ainda-podem-causar-contencao-de-locks-e-queda-de-throughput-no-postgresql-mesmo-sem-deadlocks)

---

#### 1) O que é MVCC (Multi-Version Concurrency Control) no PostgreSQL e qual problema de concorrência ele resolve na prática dentro de um sistema com múltiplas transações simultâneas?

MVCC é o mecanismo de concorrência do PostgreSQL que permite **leituras não bloqueantes**.

O banco não sobrescreve imediatamente uma linha em um `UPDATE`
Ele cria uma **nova versão da tupla** e mantém a antiga marcada com metadados de transação (`xmin` e `xmax`).

Cada transação enxerga apenas as versões **visíveis para seu snapshot** (transaction visibility).
Assim, um `SELECT` não precisa esperar um `UPDATE` terminar.

Problema resolvido: **read vs write blocking** (leitores bloqueando escritores e vice-versa).

Impacto prático:

- alta concorrência
- menos locks de leitura
- melhor throughput em APIs

Custo: surgem **dead tuples**, exigindo `VACUUM`.

#### 2) O que é uma dead tuple no PostgreSQL e por que ela é uma consequência inevitavel do MVCC?

Dead tuple é uma versão antiga de uma linha que **não é mais visível para nenhuma transação ativa**.

Ele surge porque no MVCC um `UPDATE` cria uma nova versão da tupla, marcando a antiga com `xmax`.

Quando todas as transações que poderiam enxergar aquela versão já terminaram, ela se torna "morta".

Ela continua ocupando espaço físico no heap até que o **VACUUM** a remova.

Impactos em produção:

- aumento do tamanho da tabela (bloat)
- degradação de index scan
- piora de cache hit ratio
- mais I/O

Dead tuples são consequência direta do MVCC.

#### 3) Explique como o VACUUM funciona internamente e qual a diferença entre:

- **VACUUM**
- **VACUUM FULL**
- **autovacuum**

**E quais são os trade-offs de cada um em ambiente de produção**

`VACUUM` percore a tabela identificando dead tuples e as marca como espaço reutilizável no heap, além de atualizar o visibility map.

Ele **não reduz o tamanho físico do arquivo**, apenas libera espaço interno.

`VACUUM FULL` recria fisicamente a tabela (table rewrite), remove bloat e devolve espaço ao sistema operacional, porém **bloqueia a tabela (ACCESS EXCLUSIVE LOCK)**.

`autovacuum` é o daemon automático que executa VACUUM e ANALYZE baseado em thresholds de alterações

Trade-offs:

- **VACUUM**: seguro online, mas não resolve bloat extremo
- **VACUUM FULL**: resolve bloat, porém causa downtime
- **autovacuum**: essencial em produção; se mal configurado -> degradação progressiva

VACUUM também previne **transaction ID wraparound**, um problema crítico que pode parar o banco.

#### 4) O que é o WAL (Write-Ahead Log) no PostgreSQL e por que o banco escreve primeiro no WAL antes de gravar nas tabelas?

WAL (Write-Ahead Log) é o log sequencial onde o PostgreSQL registra **todas as alterações antes de aplicá-los** nos arquivos de dados.

Regra: _write the log first, data later_

Em um `COMMIT`, o banco precisa apenas garantir que o WAL foi persistido em disco (`fsync`).
Se houver crash, o PostgreSQL faz o **replay do WAL** para restaurar consistência.

Benefícios:

- garante durabilidade (ACI**D**)
- permite crash recovery
- base para replication (streaming replicas)
- escrita sequencial (mais rápida que escrita aleatória)

Trade-off:

- aumenta I/O
- depende de configuração de `checkpoint` e `wal_buffers`

#### 5) Explique o que é um checkpoint no PostgreSQL e como ele se relaciona diretamente com o WAL e com picos de latência em produção

Checkpoint é o processo que grava em disco todas as páginas "sujas" (dirty pages) que ainda estão apenas em memória (shared buffers).

Após isso, o PostgreSQL pode descartar WAL antigo, pois os dados já estão persistidos nos _data files_.

Sem checkpoint, o recovery exigiria replay complexo do WAL.

Impacto em produção:

- muitos writes simultâneos
- spikes de I/O
- aumento de latência

Configuração inadequada (`checkpoint_timeout` e `checkpoint_completion_target`) causa **checkpoint storms**.

Trade-off:

- checkpoints frequentes -> mais I/O constante
- checkpoints raros -> recovery lento e WAL gigante

#### 6) Explique as propriedades ACID de uma transação e, principalmente, qual delas o WAL garante diretamente e qual o MVCC ajuda a implementar.

**Atomicidade:** a transação é aplicada integralmente ou revertida integralmente. Implementada via WAL e mecanismos de rollback.

**Consistência:** após o commit, o banco deve obedecer todas as constraint (FK, CHECK, UNIQUE). É garantida pelo engine + regras declaradas.

**Isolamento:** transações concorrentes não devem interferir incorretamente umas nas outras. No PostgreSQL é implementando via MVCC e isolation levels.

**Durabilidade:** após `COMMIT`, os dados sobrevivem a crash. Garantida pelo WAL com `fsync`.

WAL garante diretamente **Durabilidade e Atomicidade**.
MVCC é a base do **Isolamento**.

#### 7) Explique os níveis de isolamento no PostgreSQL e diga:

**1.** Qual é o padrão?
**2.** O PostgreSQL realmente implementa todos os níveis conforme o padrão SQL?
**3.** O que é phantom read e em qual nível ele pode ocorrer?

O nível padrão do PostgreSQL é **READ COMMITED**.

O banco não implementa exatamente todos os níveis do padrão SQL:

- `READ UNCOMMITTED` é tratado como READ COMMITED ( PostgreSQL nunca permite dirty read)
- `REPEATABLE READ` já usa snapshot isolation (mais forte que o padrão)
- `SERIALIZABLE` usa Serializable Snapshot Isolation (SSI)

Phanton read ocorre quando uma mesma query retorna novos registros inseridos por outra transação concorrente.

Ele pode ocorrer em **READ COMMITED** e é previnido em **SERIALIZABLE**.

Trade-off: quanto maior o isolamento, menor a concorrência e maior chance de abort de transação.

#### 8) O que é um deadlock no PostgreSQL, como ele acontece entre duas transações e o que o banco faz quando detecta um?

Deadlock ocorre quando duas (ou mais) transações ficam esperando locks umas das outras, criando um ciclo de dependência.

Exemplo clássico:
T1 bloqueia linha A e espera linha B.
T2 bloqueia linha B e espera linha A.

O PostgreSQL detecta o ciclo via deadlock detector e aborta uma das transações automaticamente.

Erro retornado:
`ERROR: deadlock detected`

Impacto:

- rollback inesperado
- necessidade de retry na aplicação
- pode escalar sob alta concorrência

Prevenção:

- ordem consistente de acesso a recursos
- transações curtas
- evitar locks desnecessários

#### 9) Quais são os tipos de locks no PostgreSQL (row-level, table-level e advisory) e quando cada um é usado na prática?

**Row-level locks:** aplicados automaticamente em `UPDATE`, `DELETE` e `SELECT FOR UPDATE`. Bloqueiam apenas a linha específica.

**Table-level locks:** aplicados em DDL (`ALTER TABLE`, `DROP`, `VACUUM FULL`). Podem bloquear toda a tabela dependendo do modo (ACCESS SHARE até ACCESS EXCLUSIVE).

**Advisory locks:** locks explícitos controlados pela aplicação (`pg_advisory_lock`). Não protegem dados automaticamente; servem para controle lógico.

Row locks -> concorrência fina e comum em APIs.
Table locks -> operações estruturais e manutenção.
Advisory -> controle distribuído, jobs, filas.

Trade-off: mais locks -> menos concorrência -> maior risco de deadlock.

#### 10) Qual a diferença entre:

- Index Scan
- Sequential Scan

E por que às vezes o PostgreSQL escolhe fazer um Seq Scan mesmo existindo índice?

**Sequential Scan (Seq Scan):** O PostgreSQL lê todas as páginas da tabela sequencialmente.
**Index Scan:** o banco usa o índice para localizar as linhas e depois acessa o heap para buscar os dados.

O planner escolhe baseado em **custo estimado**, não na existência do índice.

Ele pode preferir Seq Scan quando:

- grande porcentagem da tabela será lida
- tabela é pequena
- baixa seletividade do filtro
- estastísticas indicam muitas linhas correspondentes

Índice não é sempre mais rápido; acesso aleatório ao heap pode custar mais que leitura sequencial.

#### 11) O que é um EXPLAIN ANALYZE e qual a diferença entre:

- **custo estimado**
- **tempo real de execução**

**E por que isso é essencial para diagnosticar performance em produção?**

`EXPLAIN` mostra o **plano de execução estimado** pelo query planner.
`EXPLAIN ANALYZE` executa a query e mostra o **plano real com tempos medidos**.

O _cost_ é uma estimativa baseada em I/O, CPU e estástisticas — não é o tempo em ms.
O _actual time_ é o tempo real de execução.

Diferença grande entre estimado e real indica **estastisticas incorretas ou cardinalidade mal estimada**.

Usos:

- identificar seq scan inesperado
- verificar uso de índice
- encontrar join caro
- diagnosticar slow queries

É essencial em produção porque otimização sem plano real é tentativa e erro.

#### 12) Quais são os tipos principais de índices no PostgreSQL e em que tipo de dado ou consulta cada um deve ser usado?

**B-tree (padrão)**: igualdade e range(`=`, `<`, `>`, `BETWEEN`, `ORDER BY`). Usado para id, email, datas.

**Hash**: apenas igualdade (`=`). Hoje raramente necessário — B Tree normalmente substitui.

**GIN**: busca por múltiplos valores dentro de um campo. Ideal para `ARRAY`, `JSONB`, full-text search (`@>`, `?`, `tsvector`)

**GiST**: dados geométricos e buscas por proximidade (PostGIS, ranges, similaridade).

**BRIN**: tabelas gigantes com dados ordenados fisicamente (logs, séries, temporais). Muito pequeno e barato.

Escolha errada de índice pode piorar performance e aumentar custo de escrita.

#### 13) O que é um covering index (index-only scan) e qual relação ele tem com o _visibility map_ e o VACUUM?

Convering index ocorre quando a query pode ser respondida **apenas pelo índice**, sem acessar o heap (tabela).

O PostgreSQL realiza um **Index Only Scan**.

Isso só é possível se:

**1.** todas as colunas consultadas estiverem no índice <br>
**2.** a página estiver marcada como _all-visible_ no visibility map

O visibility map é atualizado pelo VACUUM.

Sem VACUUM, o banco precisa verificar o heap para cehcar visibilidade -> perde o benefício.

Resultado: VACUUM impacta diretamente performance de leitura.

#### 14) Explique o que é um partial index e um composite index, e em qual situação um índice composto na ordem errada deixa de ser utilizado pelo planner.

**Partial index** é um índice criado apenas para um subconjunto das linhas usando `WHERE`.

Exemplo:

```sql
CREATE INDEX idx_orders_open ON orders(user_id)
WHERE status = 'open';
```

Reduz tamanho do índice e custo de escrita.

**Composite index** indexa múltiplas colunas em ordem definida:

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status)
```

A ordem importa.
O planner só usa eficientemente pelo **prefixo à esquerda**.

Serve para:

- `WHERE user_id = ?`
- `WHERE user_id = ? AND status = ?`

Não serve para:

- `WHERE status = ?` (sozinho)

Escolha errada da ordem -> índice ignorado.

#### 15) Explique a diferença entre OFFSET pagination e curosr (keyset) pagination, e por que OFFSET degrada drasticamente performance em tabelas grandes.

**OFFSET pagination** usa `LIMIT ... OFFSET ...` e o banco precisa percorrer e descartar linhas anteriores.

Quanto maior o OFFSET, mais linhas o PostgreSQL lê inutilmente -> custo O(n).

**Keyset (cursor) pagination** usa uma condição baseada na última chave lida (`WHERE id > last_id`), permitindo usar índice diretamente.

OFFSET:

- simples
- mas degrada linearmente
- instável com inserções concorrentes

Keyset:

- estável
- usa index scan
- escala para tabelas grandes

Impacto: OFFSET causa alto I/O, CPU e saturação de conexões

#### 16) O que é connection pooling no PostgreSQL e por que uma aplicação Node.js pode derrubar um banco apenas abrindo muitas conexões simultâneas?

Connection pooling mantém um conjunto fixo de conexões abertas e as reutiliza entre requisições.

No PostgreSQL cada conexão cria um **backend process** no servidor.

Cada processo consome memória (work_mem, shared buffers mapping, stacks) e participa do scheduler do SO.

Muitas conexões simultâneas causam:

- alto consumo de RAM
- context switching excessivo
- queda de performance global

Node.js com muitas requisições concorrentes pode abrir centenas de conexões e saturar o banco.

O pool limita concorrência efetiva e transforma milhares de requests em poucas sessões ativas.

Ferramentas comumns: `pg.Pool` (app side) ou `PgBouncer` (server side).

#### 17) Em uma API concorrente, por que transações longas são perigosas no PostgreSQL e como elas afetam diretamente o VACUUM, MVCC e crescimento da tabela?

Uma transação longa mantém um snapshot antigo ativo.

Pelo MVCC, o PostgreSQL não pode remover versões de linhas que ainda possam ser visíveis para essa transação

Assim, o VACUUM não consegue limpar dead tuples cujo `xmin` é anterior ao snapshot.

Consequências:

- acúmulo de dead tuples
- crescimento da tabela e dos índices (bloat)
- piora de index scan e cache
- aumento de I/O

Em casos extremos pode ocorrer **transaction ID wraparound risk**.

Boas práticas:

- transações curtas
- nunca manter transação aberta aguardando I/O externo
- evitar `BEGIN` em requests HTTP longos

#### 18) O que é uma replicação (streaming replication) no PostgreSQL, o que é uma read replica, e qual problema ela resolve — e qual problema ela NÃO resolve?

Streaming replication replica continuamente o WAL do servidor primário para um servidor standby.

O standby aplica o WAL (replay) e mantém uma cópia quase em tempo real do banco.

Uma **read replica** é um standby aberto apenas para leitura.

Resolve:

- alta disponibilidade
- failover
- escalabilidade de leitura

Não resolve:

- performance de escrita (writes continuam no primário)
- queries lentas mal otimizadas

Existe atraso (replication lag), então não há consistência forte imediata.

#### 19) O que é failover no PostgreSQL e qual é o risco de perda de dados dependendo do tipo de replicação configurada?

Failover é o processo de promover uma réplica para primária após falha do servidor principal.

Na replicação assíncrona, pode haver perda de dados se o primário cair antes de enviar/aplicar o WAL na réplica.

Na replicação síncrona, o commit só é confirmado após a réplica confirmar recebimento do WAL — reduz risco de perda, mas aumenta latência.

Trade-off:

- assíncrona -> mais rápida, risco de data loss
- síncrona -> mais segura, maior latência

Failover requer mecanismos de orquestração (ex.: Patroni) e reconfiguração de clientes.

#### 20) O que são migrations com zero downtime no PostgreSQL e por que um simples `ALTER TABLE` pode derrubar uma aplicação em produção?

Zero-downtime migration é alterar o schema sem bloquear operações da aplicação.

Alguns `ALTER TABLE` existem `ACCESS EXCLUSIVE LOCK`, bloqueando SELECT/INSERT/UPDATE.

Enquanto a migration roda, todas as queries aguardam -> fila de conexões -> timeout na API.

Exemplos perigosos:

- adicionar coluna com `DEFAULT` em versões antigas.
- alterar tipo de coluna
- `VACUUM FULL`
- criar índice sem `CONCURRENTLY`

Boas práticas:

- `CREATE INDEX CONCURRENTLY`
- backfill em etapas
- deploy compatível com dois schemas
- migração + código versionados

#### 21) O que são prepared statements no PostgreSQL e por que eles podem melhorar performance — e também causar problemas quando usados incorretamente com pool de conexões?

Prepared statement é uma query previamente **parseada e planejada** pelo PostgreSQL, reutilizada com parâmetros diferentes.

O banco evita repetir:

- parse
- rewrite
- plan

Melhora latência e reduz CPU em queries frequentes.

Problema: prepared statements são ligados à sessão (conexão)

Com pool ou PgBouncer (transaction pooling), a próxima execução pode cair em outra conexão -> erro _"prepared statement does not exist_ ou plano incorreto.

Também pode ocorrer **plan caching ruim** (parameter sniffing): o planner fixa um plano não ideal para outros valores.

Uso correto:

- bom para queries muito repetidas
- cuidado com pools e PgBouncer

#### 22) O que é o problema N+1 queries, por que ORMs frequentemente causam isso, e qual o impacto real no PostgreSQL sob carga concorrente?

N + 1 ocorre quando a aplicação executa 1 query principal e depois uma query adicional para cada registro retornado.

Ex.: busca 100 usuários e faz 100 queries para buscar pedidos de cada um.

ORMs causam isso via lazy loading de relações.

Impacto:

- explosão de round-trips
- aumento de conexões ativas
- mais locks e snapshots MVCC
- saturação de pool

Solução:

- JOINs
- eager loading
- batch queries (`IN (...)`)
- agregações no banco

Não é apenas latência — degrada todo o banco sob concorrência.

#### 23) Como você implementaria idempotência em uma API usando PostgreSQL para evitar, por exemplo, dupla cobrança de pagamento?

Idempotência garante que múltiplas execuções da mesma operação produzam o mesmo resultado.

No banco, é implementado com **chave única + constraint ou idempotency key**.

Exemplo clássico:

```sql
CREATE UNIQUE INDEX idx_payment_idempotency
ON payments(idempotency_key);
```

Na API:

- primeira requisição insere
- segunda falha por unique violation
- aplicação trata o erro e retorna resultado já existente

Alternativamente:

- `INSERT ... ON CONFLICT DO NOTHING`
- ou `ON CONFLICT DO UPDATE`

UPDATE condicional sozinho não é suficiente sob concorrência real.

#### 24) Como você implementaria uma fila (queue) usando PostgreSQL, garantindo que múltiplos workers processem jobs sem pegar o mesmo item duas vezes?

A fila é implementada com uma tabela de jobs e consumo usando `SELECT ... FOR UPDATE SKIP LOCKED`.

Cada worker inicia uma transação, seleciona um job pendente, bloqueia a linha e a marca como em processamento.

`SKIP LOCKED` faz outros workers ignorarem linhas já bloqueadas.

```sql
SELECT * FROM jobs
WHERE status = 'pending'
ORDERY BY id
FOR UPDATE SKIP LOCKED
LIMIT 1
```

Isso garante que dois workers não processem o mesmo job.

Após processar: `UPDATE jobs SET status='done'`

- simples
- consistente
- limitado por throughput do banco

#### 25) O que é ANALYZE no PostgreSQL, que tipo de estatística ele coleta e por que estatísticas desatualizadas fazem o planner escolher planos ruins?

`ANALYZE` coleta estatísticas das colunas para o query planner estimar cardinalidade.

Ele amostra a tabela e registra:

- distribuição de valores (histogram)
- valores mais frequentes (MCV list)
- número de valores distintos (ndistinct)
- fração de NULLs
- correlação física

O planner usa essas estatísticas para estimar quantas linhas uma condição retornará

Se a estimativa estiver errada -> escolhe plano errado (Seq Scan vs Index Scan, Nested Loop vs Hash Join).

`autovacuum` executa autoanalyze automaticamente após certo volume de mudanças.

Estatísticas desatualizadas causam slow queries mesmo com índices corretos.

#### 26) O que é o pg_stat_activity e pg_stat_statements, e como você usaria ambos para investigar uma produção lenta?

`pg_stat_activity` mostra as conexões ativas no momento:

- query em execução
- estado (active, idle, idle in transaction)
- tempo de execução
- bloqueios

Usado para identificar:

- queries travadas
- long transactions
- lock contention
- excesso de conexões

`pg_stat_statements` agrega histórico de queries:

- tempo total
- média
- número de execuções
- rows retornadas

Usado para identificar:

- top queries mais custosas
- N+1 patterns
- queries frequentes porém lentas

`pg_stat_activity`= visão em tempo real
`pg_stat_statements` = visão estatística acumulada

#### 27) O que é PITR (Point-in-Time Recovery) no PostgreSQL e como ele utiliza o WAL para restaurar o banco até um ponto específico no tempo?

PITR (Point-in-Time Recovery) permite restaurar o banco para um instante específico.

Ele funciona combinando:

- um backup base (base backup)
- os arquivos WAL arquivados

O PostgreSQL restaura o backup e depois faz **replay do WAL** até um timestamp ou LSN definido.

Uso típico:

- recuperar deleção acidental
- erro de migration
- corrupção lógica

Sem WAL arquivado só é possível restaurar até o momento do backup.

PITR depende de `archive_mode` e `archive_command`.

#### 28) Qual a diferença entre replicação física e replicação lógica no PostgreSQL e quando você escolheria cada uma?

**Replicação física** replica os arquivos do banco a nível de página/bloco via WAL replay.

A réplica é uma cópia binária exata do primário

Usos:

- alta disponibilidade
- failover
- read replicas

**Replicação lógica** replica mudanças lógicas (INSERT/UPDATE/DELETE) por tabela/publicação.

Permite filtrar tabelas e consumir em outros sistemas.

Usos:

- migração sem downtime
- integração com outros bancos
- CDC (Change Data Capture)

Física:

- simples
- consistente
- só leitura

Lógica:

- flexível
- seletiva
- maior overhead

#### 29) Por que muitas conexões simultâneas executando transações curtas ainda podem causar contenção de locks e queda de throughput no PostgreSQL, mesmo sem deadlocks?

Mesmo transações curtas competem por:

- row-level locks
- index page locks
- WAL flush
- shared buffers
- LWLocks internas

Muitas conexões simultâneas aumentam:

- contenção em estruturas internas
- context switching
- pressão de CPU
- esperar por commit (fsync do WAL)

Mesmo sem deadlock, ocorre **lock contention e fila de espera**

Throughput não cresce linearmente com conexões — ele atinge um ponto ótimo e depois degrada.

Mais concorrência ≠ mais performance.
