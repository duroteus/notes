# Postgres

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

Regra: *write the log first, data later*

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

Após isso, o PostgreSQL pode descartar WAL antigo, pois os dados já estão persistidos nos *data files*.

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

