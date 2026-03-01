# NodeJS

## Índice

- [1) Explique, de forma objetiva, como funciona o Event...](#1-explique-de-forma-objetiva-como-funciona-o-event-loop-no-node-js-e-qual-e-o-papel-da-libuv-nesse-modelo)
- [2) Qual a diferença entre Call Stack, Task Queue (mac...](#2-qual-a-diferenca-entre-call-stack-task-queue-macrotasks-e-microtask-queue-no-node-js-e-qual-delas-tem-prioridade-de-execucao)
- [3) Por que `Promise.resolve().then()` executa antes d...](#3-por-que-promise-resolve-then-executa-antes-de-settimeout-fn-0-mesmo-com-o-tempo-zero)
- [4) O Node.js é single-threaded. Então por que operaçõ...](#4-o-node-js-e-single-threaded-entao-por-que-operacoes-como-leitura-de-arquivo-fs-readfile-ou-consultas-de-rede-nao-bloqueiam-a-aplicacao-explique-tecnicamente-o-que-realmente-acontece-por-baixo)
- [5) Qual é a diferença entre uma operação CPU-bound e...](#5-qual-e-a-diferenca-entre-uma-operacao-cpu-bound-e-i-o-bound-em-node-js-e-por-que-isso-e-critico-para-performance-do-servidor)
- [6) Se CPU-bound bloqueia o Event Loop, qual o mecanis...](#6-se-cpu-bound-bloqueia-o-event-loop-qual-o-mecanismo-do-proprio-node-js-voce-pode-usar-para-executar-esse-tipo-de-tarefa-pesada-sem-travar-a-aplicacao-e-por-que-cluster-nem-sempre-resolve-esse-problema)
- [7) Qual é a diferença arquitetural entre Processos (C...](#7-qual-e-a-diferenca-arquitetural-entre-processos-cluster-e-threads-worker-threads-em-node-js-especialmente-em-relacao-a-memoria-e-comunicacao)
- [8) Mesmo em Node.js (single thread), ainda é possível...](#8-mesmo-em-node-js-single-thread-ainda-e-possivel-ocorrer-race-condition-explique-um-cenario-real-onde-isso-pode-acontecer)
- [9) O que é backpressure em streams do Node.js e o que...](#9-o-que-e-backpressure-em-streams-do-node-js-e-o-que-acontece-com-a-aplicacao-se-voce-ignorar-isso)
- [10) Por que usar `fs.readFile` para servir arquivos gr...](#10-por-que-usar-fs-readfile-para-servir-arquivos-grandes-ex-videos-em-uma-api-http-e-uma-ma-pratica-arquitetural-em-node-js)
- [11) O que é um memory leak em Node.js e qual é um exem...](#11-o-que-e-um-memory-leak-em-node-js-e-qual-e-um-exemplo-comum-que-acontece-em-apis-http)
- [12) O que significa dizer que uma API deve ser statele...](#12-o-que-significa-dizer-que-uma-api-deve-ser-stateless-e-por-que-isso-e-importante-para-escalar-horizontalmente)
- [13) O que é idempotência em APIs HTTP e em qual tipo d...](#13-o-que-e-idempotencia-em-apis-http-e-em-qual-tipo-de-endpoint-isso-e-especialmente-importante)
- [14) Por que um endpoint `POST /payments` sem proteção...](#14-por-que-um-endpoint-post-payments-sem-protecao-de-idempotencia-pode-causar-um-problema-grave-em-producao)
- [15) O que é uma connection pool de banco de dados e po...](#15-o-que-e-uma-connection-pool-de-banco-de-dados-e-por-que-abrir-uma-nova-conexao-a-cada-requisicao-http-e-uma-pessima-ideia-em-node-js)
- [16) Qual o papel do Redis em uma arquitetura backend N...](#16-qual-o-papel-do-redis-em-uma-arquitetura-backend-node-js-e-qual-problema-ele-normalmente-resolve-melhor-do-que-o-proprio-banco-de-dados)
- [17) Explique o que é um circuit breaker e qual tipo de...](#17-explique-o-que-e-um-circuit-breaker-e-qual-tipo-de-falha-de-producao-ele-evita-em-sistemas-distribuidos)
- [18) Qual a diferença entre retry e circuit breaker e p...](#18-qual-a-diferenca-entre-retry-e-circuit-breaker-e-por-que-usar-retry-sem-cuidado-pode-piorar-uma-indisponibilidade)
- [19) O que é graceful shutdown em uma aplicação Node.js...](#19-o-que-e-graceful-shutdown-em-uma-aplicacao-node-js-rodando-em-producao-ex-kubernetes-ou-docker-e-o-que-deve-acontecer-tecnicamente-quando-o-processo-recebe-um-sigterm)
- [20) Por que usar `fs.readFileSync` (ou qualquer API `s...](#20-por-que-usar-fs-readfilesync-ou-qualquer-api-sync-dentro-de-uma-rota-http-e-considerado-uma-pratica-perigosa-em-node-js)
- [21) Qual a diferença prática entre `process.nextTick()...](#21-qual-a-diferenca-pratica-entre-process-nexttick-e-setimmediate-no-node-js-e-quando-cada-um-deve-ser-usado)
- [22) O que acontece se você criar uma Promise recursiva...](#22-o-que-acontece-se-voce-criar-uma-promise-recursiva-infinita-promise-resolve-then-loop-em-node-js-e-por-que)
- [23) Explique o que são streams no Node.js e por que el...](#23-explique-o-que-sao-streams-no-node-js-e-por-que-eles-sao-fundamentais-para-alta-performance-em-servidores)
- [24) Qual a diferença entre buffering e streaming e qua...](#24-qual-a-diferenca-entre-buffering-e-streaming-e-qual-deles-escala-melhor-em-um-servidor-node-js)
- [25) O que é rate limiting em uma API e qual problema o...](#25-o-que-e-rate-limiting-em-uma-api-e-qual-problema-operacional-real-ele-previne)
- [26) Explique a diferença entre usar fila de mensagens...](#26-explique-a-diferenca-entre-usar-fila-de-mensagens-rabbitmq-kafka-e-fazer-comunicacao-direta-http-entre-servicos-quando-a-fila-e-arquiteturalmente-necessaria)
- [27) O que é uma Dead Letter Queue (DLQ) e por que ela...](#27-o-que-e-uma-dead-letter-queue-dlq-e-por-que-ela-e-essencial-em-sistemas-que-usam-filas)
- [28) O que acontece se um consumidor de fila processa u...](#28-o-que-acontece-se-um-consumidor-de-fila-processa-uma-mensagem-executa-a-logica-com-sucesso-mas-o-processo-cai-antes-de-dar-o-ack-da-mensagem-qual-tipo-de-problema-isso-gera-no-sistema)
- [29) Qual a diferença entre at-most-once, at-least-once...](#29-qual-a-diferenca-entre-at-most-once-at-least-once-e-exactly-once-delivery-em-sistemas-de-mensageria)
- [30) Em uma API Node.js que consome uma fila de pagamen...](#30-em-uma-api-node-js-que-consome-uma-fila-de-pagamentos-por-que-idempotencia-no-consumidor-e-obrigatoria-mesmo-que-o-produtor-publique-apenas-uma-vez)
- [31) O Node.js trata exceções síncronas e erros assíncr...](#31-o-node-js-trata-excecoes-sincronas-e-erros-assincronos-de-forma-diferente-por-que-o-try-catch-nao-captura-erros-lancados-dentro-de-um-callback-assincrono)
- [32) O que é um `unhandledRejection` em Node.js e por q...](#32-o-que-e-um-unhandledrejection-em-node-js-e-por-que-ele-e-perigoso-em-producao)
- [33) Qual a diferença entre `uncaughtException` e `unha...](#33-qual-a-diferenca-entre-uncaughtexception-e-unhandledrejection-e-qual-deve-ser-a-acao-correta-da-aplicacao-ao-ocorrer-qualquer-um-deles)
- [34) Em uma API Node.js sob alta carga, o que é mais cr...](#34-em-uma-api-node-js-sob-alta-carga-o-que-e-mais-critico-para-escalabilidade-throughput-ou-latency-explique-a-diferenca-e-o-trade-off)
- [35) O que é load shedding e por que às vezes é melhor...](#35-o-que-e-load-shedding-e-por-que-as-vezes-e-melhor-rejeitar-requisicoes-do-que-aceita-las-todas)
- [36) O que é um health check e qual a diferença entre *...](#36-o-que-e-um-health-check-e-qual-a-diferenca-entre-liveness-probe-e-readiness-probe-em-ambientes-como-kubernetes)
- [37) O que deve acontecer com a readiness probe durante...](#37-o-que-deve-acontecer-com-a-readiness-probe-durante-um-graceful-shutdown-e-por-que)
- [38) Em um servidor Node.js, por que simplesmente chama...](#38-em-um-servidor-node-js-por-que-simplesmente-chamar-process-exit-ao-receber-sigterm-e-uma-pratica-perigosa)
- [39) O que significa dizer que uma API é "eventually co...](#39-o-que-significa-dizer-que-uma-api-e-eventually-consistent-e-em-que-tipo-de-arquitetura-isso-e-aceitavel-ou-necessario)
- [40) Por que tentar manter transações ACID distribuídas...](#40-por-que-tentar-manter-transacoes-acid-distribuidas-entre-microservicos-geralmente-e-considerado-uma-ma-ideia-arquitetural)
- [41) O que é o padrão Saga em microserviços e qual prob...](#41-o-que-e-o-padrao-saga-em-microservicos-e-qual-problema-ele-resolve)
- [42) Por que colocar operações pesadas (ex.: envio de e...](#42-por-que-colocar-operacoes-pesadas-ex-envio-de-email-geracao-de-pdf-integracao-externa-diretamente-dentro-de-uma-rota-http-e-uma-ma-pratica-arquitetural-em-node-js)
- [43) Em uma API escalável, quando você deveria retornar...](#43-em-uma-api-escalavel-quando-voce-deveria-retornar-http-202-accepted-em-vez-de-200-ok)
- [44) Se você tivesse que projetar um endpoint de geraçã...](#44-se-voce-tivesse-que-projetar-um-endpoint-de-geracao-de-relatorio-pesado-qual-seria-o-fluxo-correto-da-requisicao-ate-o-cliente-obter-o-arquivo-final)
- [45) O que acontece com seu banco de dados se você impl...](#45-o-que-acontece-com-seu-banco-de-dados-se-voce-implementar-cache-com-redis-mas-nao-usar-ttl-expiracao-nas-chaves)
- [46) O que é o problema de cache stampede (ou thunderin...](#46-o-que-e-o-problema-de-cache-stampede-ou-thundering-herd-e-como-voce-mitigaria-isso-em-node-js-usando-redis)
- [47) Por que usar apenas `await` em todas as operações...](#47-por-que-usar-apenas-await-em-todas-as-operacoes-dentro-de-uma-rota-pode-reduzir-drasticamente-a-performance-de-uma-api-node-js)
- [48) Quando não é seguro usar `Promise.all` dentro de u...](#48-quando-nao-e-seguro-usar-promise-all-dentro-de-uma-rota)
- [49) Por que simplesmente aumentar o número de instânci...](#49-por-que-simplesmente-aumentar-o-numero-de-instancias-node-horizontal-scaling-nao-resolve-gargalos-causados-por-banco-de-dados)
- [50) Qual a diferença entre read replica e sharding, e...](#50-qual-a-diferenca-entre-read-replica-e-sharding-e-em-que-cenario-cada-um-e-utilizado)
- [51) Se você tivesse investigando uma API Node.js lenta...](#51-se-voce-tivesse-investigando-uma-api-node-js-lenta-em-producao-quais-metricas-voce-analisaria-primeiro-para-identificar-gargalo)
- [52) O que é event loop lag e por que ele é um dos sina...](#52-o-que-e-event-loop-lag-e-por-que-ele-e-um-dos-sinais-mais-importantes-de-problema-em-uma-aplicacao-node-js)
- [53) Resuma o princípio fundamental para escalar aplica...](#53-resuma-o-principio-fundamental-para-escalar-aplicacoes-node-js-em-producao)

---

#### 1) Explique, de forma objetiva, como funciona o Event Loop no Node.js e qual é o papel da libuv nesse modelo.

O Node.js executa JavaScript em uma única thread principal. O Event Loop é o mecanismo que coordena a execução: ele processa a Call Stack, verifica filas de callbacks e decide quando cada tarefa pode rodar sem bloquear a thread.

Quando uma operação assíncrona é iniciada (ex.: fs, DNS, criptografia), ela é delegada para a libuv. A libuv interage com o sistema operacional para monitorar I/O e, quando necessário, utiliza um thread pool interno para tarefas bloqueantes.

Ao terminar a operação, a libuv coloca o callback em uma fila do Event Loop. O loop então agenda esse callback para execução quando a Call Stack estiver vazia.

Esse modelo permite I/O non-blocking com apenas uma thread de JavaScript.

---

#### 2) Qual a diferença entre Call Stack, Task Queue (macrotasks) e Microtask Queue no Node.js? E qual delas tem prioridade de execução?

A Call Stack é onde o JavaScript executa código síncrono. Enquanto houver funções na pilha, o Event Loop não processa callbacks assíncronos.

Callbacks assíncronos são colocados em filas. Macrotasks incluem timers, I/O, setImmediate. Microtasks incluem resoluções de Promise e queueMicrotask.

Ao final de cada ciclo do EventLoop, quando a Call Stack esvazia, o Node primeiro executa todas as microtasks pendentes e só depois executa a próxima macrotask.

Portanto, a Microtask Queue tem prioridade sobre a tasks Queue

 - **Call Stack:** é a pilha onde o JavaScript executa funções síncronas. Enquanto existir função nela, nada assíncrono roda.
 - **Task Queue (macrotasks):** fila de callbacks de eventos como `setTimeout`, `setInterval`, I/O, `setImmediate`.
 - **Microtask Queue:** fila mais prioritária, usada por `Promise.then/catch/finally` e `queueMicrotask`.

**Fluxo real:**

**1.** Executa tudo da Call Stack
**2.** Executa **todas as microtasks**
**3.** Depois pega **uma macrotask**
**4.** Repete o ciclo

Ou seja: **microtask sempre roda antes da próxima macrotask.**

---

#### 3) Por que `Promise.resolve().then()` executa antes de `setTimeout(fn, 0)` mesmo com o tempo zero?

`Promise.then` agenda seu callback na Microtask Queue, enquanto `setTimeout` agenda na fila de macrotasks (fase de timers do Event Loop).

Quando a Call Stack esvazia, o Event Loop executa todas as microtasks pendentes antes de avançar para qualquer fase de macrotask.

Portanto, mesmo com o delay 0, o `setTimeout` só será executado no próximo ciclo do loop.

Como microtasks têm prioridade e são drenadas imediatamente após a stack esvaziar, o callback da Promise sempre executa antes do timer.

---

#### 4) O Node.js é single-threaded. Então por que operações como leitura de arquivo (`fs.readFile`) ou consultas de rede não bloqueiam a aplicação? Explique técnicamente o que realmente acontece por baixo.

O JavaScript do Node roda em uma única thread, mas o runtime utiliza a libuv para realizar I/O assíncrono.

Quando chamamos `fs.readFile` ou uma operação de rede, o Node não executa isso na Call Stack. A operação é delegada para a libuv.

Para sockets de rede, a libuv registra a operação no sistema operacional usando mecanismos de notificação (epoll/kqueue/IOCP).
Para tarefas bloqueantes, como acesso a arquivos ou criptografia, ela usa um thread pool interno.

Quando a operação termina, a libuv coloca o callback em uma fila do Event Loop, e ele é executado quando a Call Stack estiver livre.
Assim o Node consegue non-blocking I/O mesmo com apenas uma thread de JavaScript.

---

#### 5) Qual é a diferença entre uma operação CPU-bound e I/O-bound em Node.js, e por que isso é crítico para performance do servidor?

CPU-bound é uma tarefa cujo tempo é gasto principalmente em processamento de CPU, como loops pesados, criptografia ou parsing grande.
Como o Node executa JavaScript em uma única thread, esse tipo de tarefa bloqueia o Event Loop e impede o servidor de atender outras requisições.

I/O-bound é uma tarefa dominada por espera externa, como rede, banco de dados ou disco.
O Node delega essas operações à libuv e continua processando outras requisições enquanto aguarda o resultado.

Isso é crítico porque Node é otimizado para alta concorrência de I/O.
Se colocarmos tarefas CPU-bound na thread principal, derrubamos a escalabilidade do servidor mesmo com poucos usuários.

---

#### 6) Se CPU-bound bloqueia o Event Loop, qual o mecanismo do próprio Node.js você pode usar para executar esse tipo de tarefa pesada sem travar a aplicação? E Por que Cluster nem sempre resolve esse problema?

Para tarefas CPU-bound em Node.js, usamos **Worker Threads**, que permitem executar JavaScript em threads separadas dentro do mesmo processo.
Isso evita bloquear o Event Loop da thread principal.

Workers compartilham memória opcionalmente (SharedArrayBuffer) e comunicam-se via message passing.

Já o **Cluster** cria múltiplos processos Node independentes, geralmente um por CPU core.
Ele melhora o paralelismo de requisições, mas **não resolve bloqueio dentro de uma requisição específica**.
Se uma rota fizer processamento pesado, ela continuará bloqueando o Event Loop daquele worker.

> **Cluster escala throughput; Worker Threads resolvem CPU-bound interno.**

---

#### 7) Qual é a diferença arquitetural entre Processos (Cluster) e Threads (Worker Threads) em Node.js, especialmente em relação a memória e comunicação?

Cluster cria múltiplos processos Node independentes. Cada processo possui sua própria memória (heap e V8 isolado) e um Event Loop próprio.
A comunicação entre eles ocorre via IPC (message passing) e não há compartilhamento direto de memória.

Worker Threads executam dentro do mesmo processo Node. Elas possuem stacks separadas, mas podem compartilhar memória usando `SharedArrayBuffer` ou `Atomics`, além de message passing.

Processos são mais isolados e seguros, porém mais caros em memória.
Threads são mais leves e ideais para CPU-bound, pois evitam serialização de grandes dados entre processos.

---

#### 8) Mesmo em Node.js (single thread), ainda é possível ocorrer race condition. Explique um cenário real onde isso pode acontecer.

Mesmo sendo single-threaded, Node pode ter race condition quando múltiplas requisições acessam e modificam o mesmo recurso assíncrono.

Exemplo: dois usuários chamam simultaneamente uma rota de saque. Ambas as requisições leem o saldo no banco (`SELECT balance`), recebem 100, validam que é suficiente e depois fazem `UPDATE balance = 50`.

Como houve uma janela entre leitura e escrita, as duas operações são aprovadas e o saldo final fica incorreto.

O problema não é thread, mas concorrência lógica entre operações assíncronas.
Soluções incluem lock distribuído (ex.: Redis), transações ou operações atômicas no banco.

---

#### 9) O que é backpressure em streams do Node.js e o que acontece com a aplicação se você ignorar isso?

Backpressure é o mecanismo de controle de fluxo das streams do Node.js.
Ele ocorre quando um `Readable` produz dados mais rápido do que um `Writable` consegue consumir.

Quando o buffer do consumidor enche, o método `.write()` retorna `false`, sinalizando ao produtor para pausar.
A leitura só deve continuar após o evento `drain`.

Se ignorarmos o backpressure, os buffers crescem continuamente, aumentando o uso de memória e pressionando o garbage collector, podendo causar alta latência ou crash por falta de memória.

Streams e `.pipe()` já implementam esse controle automaticamente.

---

#### 10) Por que usar `fs.readFile` para servir arquivos grandes (ex.: videos) em uma API HTTP é uma má prática arquitetural em Node.js?

`fs.readFile` carrega todo o arquivo na memória antes de enviar a resposta.
Para arquivos grandes ou mútiplcas requisições simultâneas, isso consome muita RAM, pressiona o garbage collector e pode derrubar o processo por falta de memória.

Node é otimizado para streaming, não para buffering.
O correto é usar `fs.createReadStream`e enviar em chunks, permitindo envio progressivo e controle de backpressure.

Streams mantêm baixo uso de memória e melhoram escalabilidade do servidor.

---

#### 11) O que é um memory leak em Node.js e qual é um exemplo comum que acontece em APIs HTTP?

Memory leak é quando objetos permanecem referenciados e não podem ser coletados pelo Garbage Collector, fazendo o uso de memória crescer continuamente ao longo do tempo.

Em Node isso costuma ocorrer por retenção de estado global.
Um exemplo comum é implementar cache em memória sem expiração, armazenando respostas de requisições em um objeto global.

Como as referênciuas nunca são removidas, o heap cresce, o GC executa cada vez mais e a aplicação sofre degradação de performance ou crash por OOM.

---

#### 12) O que significa dizer que uma API deve ser stateless e por que isso é importante para escalar horizontalmente?

Uma API stateless é aquela em que cada requisição contém todas as informações necessárias para ser processada, sem depender de estado armazenado em memória do servidor.

O servidor não guarda sessão local do usuário; autenticação e contexto vêm na própria requisição (ex.: token/JWT, headers, cookies assinados ou dados em banco/Redis).

Isso permite escalar horizontalmente porque qualquer instância pode atender qualquer requisição.
O load balancer pode distribuir camadas livremente sem precisar de "sticky session".

Se o servidor guardar estado em memória, a requiisição precisaria sempre cair na mesma instância, dificultando escalabilidade e tolerância a falhas.


---

#### 13) O que é idempotência em APIs HTTP e em qual tipo de endpoint isso é especialmente importante?

Idempotência é a propriedade de uma operação que, quando executada repetidamente com os mesmos parâmetros, não altera o estado final além da primeira execução.

Em HTTP, métodos como PUT e DELETE devem ser idempotentes.
Por exemplo, deletar um recurso duas vezes deve resultar no mesmo estado: o recurso continua inexistente, mesmo que a resposta mude.

Isso é especialmente importante em sistemas distribuídos, porque retries automáticos podem ocorrer devido a timeout ou falhas de rede.
A idempotência evita feitos duplicados, como cobranças ou criação repetida de pedidos.

---

#### 14) Por que um endpoint `POST /payments` sem proteção de idempotência pode causar um problema grave em produção?

Um `POST /payments` cria um novo recurso e normalmente não é idempotente.
Se houver timeout, perda de conexão ou retry automático do cliente/load balancer, a mesma requisição pode ser enviada novamente.

Como o servidor não sabe que é a mesma operação, ele processa duas vezes e gera cobranças duplicadas.
O cliente pode não ter recebido a primeira resposta, mas o pagamento já foi efetuado.

Por isso sistemas de pagamento usam **idempotency keys** (ex.: header único por operação) para garantir que múltiplas tentativas resultem em apenas uma transação.

---

#### 15) O que é uma connection pool de banco de dados e por que abrir uma nova conexão a cada requisição HTTP é uma péssima ideia em Node.js?

Um connection pool é um conjunto de conexões de banco previamente abertas e reutilizadas pelas requisições da aplicação.

Abrir uma conexão a cada request é caro porque envolve handshake TCP, autenticação e negociação do protocolo do banco, aumentando latência e consumo de CPU.

Além disso, bancos possuem limite de conexões simultâneas. Criar conexões por requisição rapidamente esgota esse limite, causando filas, timeouts e falhas.

O pool mantém um número controlado de conexões abertas e as reutiliza, reduzindo latência e estabilizando o uso de recursos tanto na aplicação quanto no banco.

---

#### 16) Qual o papel do Redis em uma arquitetura backend Node.js e qual problema ele normalmente resolve melhor do que o próprio banco de dados?

Redis é um data store in-memory usado principalmente como camada de cache em arquiteturas Node.js.

Ele armazena dados frequentemente acessados para evitar consultas repetidas ao banco relacional, reduzindo latência e principalmente a carga de leitura no banco.

Bancos relacionais são bons para consistência e queries complexas, mas escalam pior sob alto volume de leitura.
O Redis absorve essas leituras, protegendo o banco de sobrecarga.

Além de cache, também é usado para sessões, locks distribuídos, rate limiting e filas leves.

---

#### 17) Explique o que é um circuit breaker e qual tipo de falha de produção ele evita em sistemas distribuídos.

Circuit breaker é um padrão de resiliência que monitora falhas ao chamar um serviço externo.
Após um número de erros consecutivos, ele "abre" o circuito e passa a falhar rapidamente sem tentar novas chamadas.

Depois de um tempo, entra em estado half-open e permite algumas requisições de teste.
Se funcionarem, fecha novamente; se falharem, reabre.

Isso evita bloquear threads e conexões esperando um serviço indisponível e previne falhas em cascata que poderiam derubar todo o sistema.

---

#### 18) Qual a diferença entre retry e circuit breaker e por que usar retry sem cuidado pode piorar uma indisponibilidade?

Retry é a repetição automática de uma operação após uma falha transitória, geralmente com backoff.

Circuit breaker é um mecanismo de proteção: após detectar falhas consecutivas, ele interrompe novas chamadas de serviço por um período e falha rapidamente.

O retry tenta novamente; o circuit breaker decide não tentar.

Usar retry sem controle pode piorar a indisponibilidade, pois múltiplos clientes passam a reenviar requisições simultaneamente, sobrecarregando ainda mais um serviço já degradado (retry storm). Por isso retries devem usar backoff exponencial e, idealmente, trabalhar junto com circuit breaker.

---

#### 19) O que é graceful shutdown em uma aplicação Node.js rodando em produção (ex.: Kubernetes ou Docker) e o que deve acontecer tecnicamente quando o processo recebe um `SIGTERM`?

Graceful shutdown é o processo controlado de encerramento da aplicação sem interromper requisições em andamento.

Ao receber `SIGTERM`, o servidor deve parar de aceitar novas conexões (ex.: `server.close()`), mas continuar processando as requisições já abertas.

Durante esse período, deve finalizar jobs em andamento, fechar conexões com banco/Redis, encerrar workers e liberar recursos.

Somente após concluir ou atingir timeout seguro o processo deve chamar o `process.exit`.
Isso evita perda de dados, respostas truncadas e inconsistência em ambientes com load balancer ou Kubernetes.

---

#### 20) Por que usar `fs.readFileSync` (ou qualquer API `sync`) dentro de uma rota HTTP é considerado uma prática perigosa em Node.js?

APIs síncronas em Node.js bloqueiam o Event Loop.
Durante a execução de um `fs.readFileSync`, a thread principal fica aguardando o I/O terminar e não pode atender outras requisições.

Como Node depende de concorrência assíncrona para escalar, uma operação bloqueante faz todas as conexões aguardarem, causando aumento de latência e timeouts

Em rotas HTTP devemos sempre usar APIs assíncronas ou streams para não bloquear o servidor.

---

#### 21) Qual a diferença prática entre `process.nextTick()` e `setImmediate()` no Node.js e quando cada um deve ser usado?

`process.nextTick()` executa imediatamente após a função atual terminar, antes de qualquer fase do Event Loop, inclusive antes de Promises e I/O.

`setImmediate()` agenda a execução para a próxima iteração do Event Loop, na fase check, permitindo que operações de I/O sejam processadas primeiro.

`nextTick` tem prioridade muito alta e pode causar starvation se usado em excesso.
`setImmediate` é mais seguro para adiar trabalho sem bloquear o servidor.

---

#### 22) O que acontece se você criar uma Promise recursiva infinita (`Promise.resolve().then(loop)`) em Node.js e por quê?

Promises usam a Microtask Queue, que tem prioridade máxima no Event Loop. O Node executa todas as microtasks antes de processar timers ou I/O.

Uma promise recursiva infinita agenda sempre uma nova microtask antes do loop avançar. Como a fila nunca esvazia, o Event Loop nunca chega às fases de I/O.

O resultado é starvation: o servidor permanece ativo, porém não processa requisições nem sockets.

---

#### 23) Explique o que são streams no Node.js e por que eles são fundamentais para alta performance em servidores.

Streams são abstrações para leitura e escrita de forma incremental (chunks), sem carregar todo o conteúdo em memória.

Em vez de bufferizar um arquivo ou resposta inteira, o Node processa e transmite dados conforme ficam disponíveis.
Isso reduz drasticamente uso de RAM e permite envio progressivo ao cliente.

Streams também implementam backpressure: o produtor desacelera quando o consumidor não consegue acompanhar.

Elas são fundamentais porque HTTP, arquivos, sockets e compressão no Node são baseados em streams, permitindo alta concorrência com baixo consumo de recursos.

---

#### 24) Qual a diferença entre buffering e streaming e qual deles escala melhor em um servidor Node.js?

Buffering é quando todo o conteúdo é carregado em memória antes de ser processado ou enviado (ex.: `fs.readFile`).
Streaming processa e envia dados em pequenos chunks conforme ficam disponíveis.

Com buffering, cada requisição consome muita RAM e só responde após terminar a leitura completa.
Com streaming, a resposta começa imediatamente e o uso de memória permanece constante.

Streaming escala melhor em Node.js porque permite atender muitas conexões simultâneas com baixo consumo de memória e aproveita o backpressure do Event Loop.

---

#### 25) O que é rate limiting em uma API e qual problema operacional real ele previne?

Rate limiting é a limitação do número de requisições que um cliente pode fazer em um intervalo de tempo.

Ele previne sobrecarga do servidor causada por loops de cliente, scraping, bots ou ataques de negação de serviço (DoS).
Também protege recursos internos como banco de dados e APIs externas de serem saturados.

Sem rate limiting, poucos clientes podem consumir toda a capacidade do sistema e causar indisponibilidade para usuários legitimos.

---

#### 26) Explique a diferença entre usar fila de mensagens (RabbitMQ/Kafka) e fazer comunicação direta HTTP entre serviços. Quando a fila é arquiteturalmente necessária?

Comunicação HTTP entre serviços é síncrona: o serviço chamador precisa que o outro esteja disponível e respondendo dentro do timeout. Isso acopla disponibilidade e latência entre serviços.

Filas de mensagens são comunicação assíncrona. O produtor publica uma mensagem e continua sem esperar o consumidor processar.
A fila atua como buffer, absorvendo picos de carga e permitindo processamento posterior.

Isso aumenta resiliência: se o consumidor cair, as mensagems permanecem e serão processadas depois.

Filas são necessárias quando o processamento pode ser eventual, quando há tarefas demoradas (emails, processamento, intregações externas) ou quando múltiplos serviços precisam reagir ao mesmo evento sem depender diretamente um dos outros.

---

#### 27) O que é uma Dead Letter Queue (DLQ) e por que ela é essencial em sistemas que usam filas?

Dead Letter Queue é uma fila para onde são enviadas mensagens que falharam repetidamente no processamento ou violaram alguma regra (ex.: exceder número de retries ou TTL).

Ela evita que uma mensagem "envenenada" fique sendo reprocessada infinitamente e bloqueie o consumidor.

A DLQ permite inspeção, audiotira e tratamento manual ou reprocessamento controlado.

Sem DLQ, erros persistentes podem travar a fila inteira ou causar loops de retry.

---

#### 28) O que acontece se um consumidor de fila processa uma mensagem, executa a lógica com sucesso, mas o processo cai antes de dar o ACK da mensagem? Qual tipo de problema isso gera no sistema?

Se o consumidor processa a mensagem mas cai antes de enviar o ACK, o broker considera que ela não foi processada.

A mensagem será reenviada para outro consumidor ou para o mesmo quando ele voltar.
Isso gera **processamento duplicado**.

Esse comportamento faz parte do modelo *at-least-once delivery*: a fila garante que a mensagem não será perdida, mesmo que precise entregar mais de uma vez.

Por isso consumidores devem ser idempotentes ou usar deduplicação (ex.: idempotency key, controle em banco, Redis).

---

#### 29) Qual a diferença entre at-most-once, at-least-once e exactly-once delivery em sistemas de mensageria?

**At-most-once**: a mensagem é entregue no máximo uma vez; pode ser perdida, mas nunca duplicada.

**At-least-once**: a mensagem é entregue uma ou mais vezes; garante que não será perdida, porém pode haver duplicidade se ocorrer falha antes do ACK.

**Exactly-once**: a mensagem é processada exatamente uma vez; exige coordenação transacional e deduplicação, sendo dificil e custoso em sistemas distribuídos.

Por isso, a maioria dos sistemas usa at-least-once e exisge consumidores idempotentes.

---

#### 30) Em uma API Node.js que consome uma fila de pagamentos, por que idempotência no consumidor é obrigatória mesmo que o produtor publique apenas uma vez?

Mesmo que o produtor publique apenas uma vez, o broker pode reenviar a mensagem se o consumidor cair antes do ACK.

Isso ocorre porque a maioria das filas usa *at-least-once delivery*, proiorizando não perder mensagens.

Sem idempotência, o mesmo pagamento poderia ser processado duas vezes, gerando cobranças duplicadas.

O consumidor deve registrar um identificador único da operação (ex.: paymentId ou idempotency key) e garantir que múltiplos processamentos resultem no mesmo estado final.

---

#### 31) O Node.js trata exceções síncronas e erros assíncronos de forma diferente. Por que o `try/catch` não captura erros lançados dentro de um callback assíncrono?

`try/catch` só captura exceções lançadas dentro do mesmo fluxo síncrono da Call Stack.

Em operações assíncronas, o callback executa em outro ciclo do Event Loop.
Quando ele roda, a função de continha o `try/catch` já terminou e não está mais na stack.

Por isso o erro não é interceptado.
Erros assíncronos devem ser tratados via callbacks de erro, `Promise.catch` ou `async/await` com `try/catch`.

---

#### 32) O que é um `unhandledRejection` em Node.js e por que ele é perigoso em produção?

`unhandledRejection` ocorre quando uma Promise é rejeitada e não há `.catch()` nem tratamento via `await`/`try-catch`.

Isso é perigoso porque a aplicação pode continuar rodando após um erro crítico, com recursos parcialmente inicializados, conexões quebradas ou dados inconsistentes.

Versões recentes do Node podem encerrar o processo, mas não se deve depender disso.
Em produção devemos sempre registrar `process.on('unhandledRejection')`, logar o erro e iniciar o shutdown controlado.

---

#### 33) Qual a diferença entre `uncaughtException` e `unhandledRejection`, e qual deve ser a ação correta da aplicação ao ocorrer qualquer um deles?

`uncaughtException` é um erro síncrono não capturado que chegou ao topo da stack.

`unhandledRejection` é uma Promise rejeitada sem tratamento.

Ambos indicam que a aplicação entrou em estado inconsistente.
A prática recomendada é registrar o erro, iniciar graceful shutdown e encerrar o processo, permitindo que o orquestrador reinicie.

Continuar executando após erro fatal pode causar corrupção de estado e comportamento imprevisível.

---

#### 34) Em uma API Node.js sob alta carga, o que é mais crítico para escalabilidade: throughput ou latency? Explique a diferença e o trade-off.

Latency é o tempo que uma requisição individual leva para receber resposta.
Throughput é a quantidade de requisições processadas por unidade de tempo.

Há trade-off: otimizar throughput geralmente envolve filas e processamento em lote, o que aumenta a latência.
Otimizar latência exige respostas imediatas, reduzindo a capacidade total sob carga.

APIs interativas (ex.: login, checkout) priorizam baixa latência.
Processamentos assíncronos ou analytics priorizam alto throughput.

Escalabilidade não é apenas suportar mais requisições, mas manter latência aceitável sob carga.

---

#### 35) O que é load shedding e por que às vezes é melhor rejeitar requisições do que aceitá-las todas?

Load shedding é a prática de rejeitar requisições deliberadamente quando o sistema atinge sua capacidade máxima.

Sem isso, o servidor aceita mais trabalho do que consegue processar, acumulando filas, aumentando latência e causando timeouts em massa, podendo derrubar dependências como banco de dados.

Ao retornar erro rápido (ex.: HTTP 503), o sistema preserva recursos e continua atendendo parte dos usuários.

É melhor falhar parcialmente de forma controlada do que falhar completamente.

---

#### 36) O que é um health check e qual a diferença entre *liveness probe* e *readiness probe* em ambientes como Kubernetes?

Health check é um endpoint usado por load balancers ou orquestradores para verificar o estado da aplicação.

**Liveness probe** verifica se o processo está vivo.
Se falhar, o orquestrador reinicia o container/processo.

**Readiness probe** verifica se a aplicação está pronta para receber requisições.
Se falhar, o tráfego é removido, mas o processo não é reiniciado.

Exemplo: durante startup, migrações ou perda de conexão com banco, a aplicação pode não estar pronta (readiness false) mas ainda viva (liveness true)

---

#### 37) O que deve acontecer com a readiness probe durante um graceful shutdown e por quê?

Durante o graceful shutdown a aplicação deve marcar a readiness probe como false.

Isso sinaliza ao load balancer/orquestrador para parar de enviar novas requisições à instância.
Enquanto isso, o servidor continua processando as conexões já abertas até finalizá-las.

Depois de concluir as requisições em andamento e fechar recursos (DB, Redis, filas), o processo pode encerrar com segurança.

---

#### 38) Em um servidor Node.js, por que simplesmente chamar `process.exit()` ao receber `SIGTERM` é uma prática perigosa?

Chamar `process.exit()` encerra o processo imediatamente, sem aguardar requisições em andamento, conexões abertas ou operações assíncronas pendentes.

Isso pode resultar em respostas truncadas, transações incompletas e inconsistência de dados.

No graceful shutdown, o servidor deve primeiro parar de aceitar novas conexões, aguardar as requisições ativas finalizarem, fechar recursos (banco, Redis, filas) e só então encerrar o processo.

Encerrar abruptamente compromete confiabilidade e integridade do sistema.

---

#### 39) O que significa dizer que uma API é "eventually consistent" e em que tipo de arquitetura isso é aceitável ou necessário?

***Eventually consistent*** significa que, após uma operação, o sistema pode apresentar estados temporariamente diferentes entre serviços, mas todos convergem para o mesmo estado final após propagação dos eventos.

Isso ocorre em arquiteturas distribuídas e orientadas a eventos, onde o processamento é assíncrono via filas ou múltiplos serviços.

É aceitável quando a resposta imediata não precisa refletir todas as consequências da operação, como envio de email, atualização de cache ou projeção de dados.

Permite maior escalabilidade e desacoplamento em troca de consistência imediata.

---

#### 40) Por que tentar manter transações ACID distribuídas entre microserviços geralmente é considerado uma má ideia arquitetural?

Transações ACID distribuídas exigem coordenação entre múltiplos serviços, normalmente via two-phase commit (2PC).

Isso cria forte acoplamento, dependência de disponibilidade simultânea e locks mantidos através da rede.
Em caso de falha parcial ou timeout, serviços podem ficar bloqueados aguardando commit/rollback.

O resultado é alta latência, baixa disponibilidade e risco de deadlock em escala.

Por isso arquiteturas de microserviços preferem consistência eventual com eventos, compensações (sagas) e operações idempotentes em vez de transações distribuídas.

---

#### 41) O que é o padrão Saga em microserviços e qual problema ele resolve?

Saga é um padrão para gerenciar consistência em microserviços sem usar transações distribuídas.

A operação é dividida em várias transações locais independentes.
Se uma etapa falhar, são executadas ações compensatórias para desfazer os efeitos anteriores.

Isso substitui o rollback ACID por compensação de negócio.

O padrão resolve o problema de manter consistência em arquiteturas distribuídas onde não é viável usar two-phase commit.

---

#### 42) Por que colocar operações pesadas (ex.: envio de email, geração de PDF, integração externa) diretamente dentro de uma rota HTTP é uma má prática arquitetural em Node.js?

Operações pesadas dentro da rota fazem o tempo da requisição depender de tarefas demoradas ou instáveis (integrações, geração de arquivos, envio de email).

Isso mantém conexões abertas por muito tempo, aumenta latência, consome workers/conexões e reduz throughput do servidor.

Além disso, se o cliente cancelar ou houver timeout, o trabalho é perdido ou fica inconsistente.

O correto é responder rápido (ex.: 202 Accepted), publicar em uma fila e processar de forma assíncrona em um worker.

Isso desacopla disponibilidade, melhora escalabilidade e aumenta resiliência.

---

#### 43) Em uma API escalável, quando você deveria retornar HTTP 202 Accepted em vez de 200 OK?

`200 OK` indica que a requisição foi processada e o resultado já está disponível na resposta.

`202 Accepted` indica que o servidor apenas aceitou a requisição para processamento posterior, geralmente assíncrono (ex.: enfileirado), e o resultado ainda não está pronto.

É usado para tarefas longas ou background jobs, como geração de relatórios, envio de email ou processamento de mídia.

Normalmente o cliente recebe um identificador para consultar o status depois.

---

#### 44) Se você tivesse que projetar um endpoint de geração de relatório pesado, qual seria o fluxo correto da requisição até o cliente obter o arquivo final?

O endpoint recebe a requisição e retorna `202 Accepted` com um `jobId`.
A API registra o job no banco (status: `pending`) e publica a tarefa em uma fila.

Um worker consome a fila, gera o relatório e armazena o arquivo em storage (ex.: S3/objeto), atualizando o status para `completed` ou `failed`.

O cliente consulta `/reports/{jobId}` para verificar status (polling ou webhook).
Quando pronto, a API retorna uma URL de download do arquivo.

Esse fluxo desacopla HTTP do processamento pesado e permite escalar workers independente da API.

---

#### 45) O que acontece com seu banco de dados se você implementar cache com Redis mas não usar TTL/expiração nas chaves?

Sem TTL, o cache cresce indefinidamente e pode armazenar dados desatualizados por tempo indefinido.

Quando a memória do Redis atinge o limite, ele começa a remover chaves por política de eviction, possivelmente apagando dados importantes de forma imprevisível.

Isso pode causar cache inconsistente e picos de carga no banco (cache miss em massa).
TTL garante renovação natural dos dados e mantém o cache saudável e previsível.

---

#### 46) O que é o problema de cache stampede (ou thundering herd) e como você mitigaria isso em Node.js usando Redis?

Cache stampede ocorre quando uma chave muito acessada expira e múltiplas requisições simultâneas tentam recomputar o valor, sobrecarregando o banco.

A mitigação comum é usar lock distribuido no Redis para permitir que apenas uma requisição regenere o cache, enquanto as demais aguardam ou recebem valor antigo.

Outras estratégias incluem *stale-while-revalidate* e adicionar *jitter* ao TTL para evitar expiração simultânea.

---

#### 47) Por que usar apenas `await` em todas as operações dentro de uma rota pode reduzir drasticamente a performance de uma API Node.js?

`await` executado sequencialmente faz operações I/O independentes rodarem uma após a outra, aumentando a latência total da requisição.

Como chamadas a banco ou APIs externas são I/O-bound, elas podem ser aguardadas em paralelo.

Usar `Promise.all` permite disparar todas simultâneamente e aguardar apenas o tempo da mais lenta.

O uso inadequado de `await` reduz concorrência e degrada performance da API sem aumentar carga de CPU.

---

#### 48) Quando não é seguro usar `Promise.all` dentro de uma rota?

Não é seguro usar `Promise.all` quando as operações possuem dependência entre si, ou seja, o resultado de uma é necessário para iniciar a outra.

Também deve ser evitado quando as operações geram efeitos colaterais (ex.: múltiplos writes, pagamentos, envios) que não podem ocorrer simultâneamente.

Outro caso é quando há limite de recursos externos (pool de conexões, rate limit de API). Disparar muitas operações paralelas pode saturar banco ou serviço externo.

Nesses cenários, é melhor executar sequencialmente ou usar controle de concorrência (fila ou limiter).

---

#### 49) Por que simplesmente aumentar o número de instâncias Node (horizontal scaling) não resolve gargalos causados por banco de dados?

Se o gargalo está no banco, aumentar instâncias Node apenas aumenta o número de requisições concorrentes competindo pelo mesmo banco.

Isso gera mais conexões abertas, mais locks, mais contenção de CPU e I/O no banco, podendo piorar latência e causar timeout.

Escalar a camada stateless (Node) não resolve limitações da camada stateful (banco).
A solução envolve otimização de quieries, indices, caching, read replicas ou particionamento.

Escala horizontal só resolve gargalos quando o limite está na camada da aplicação, não na dependência.

---

#### 50) Qual a diferença entre read replica e sharding, e em que cenário cada um é utilizado?

Read replica é uma cópia do banco principal usada apenas para consultas de leitura.
Ela reduz carga do banco primário e melhora throughput de reads, mantendo um único ponto de escrita.

Sharding divide os dados horizontalmente entre múltiplos bancos, onde cada shard contém apenas uma parte do dataset.
Isso permite escalar tanto leitura quanto escrita.

Read replica é usada quando o problema é excesso de leitura.
Sharding é necessário quando o volume total de dados ou escrita ultrapassa a capacidade de um único banco.

---

#### 51) Se você tivesse investigando uma API Node.js lenta em produção, quais métricas você analisaria primeiro para identificar gargalo?

Primeiro analisaria latência por endpoint (p95/p99) para identificar quais rotas estão lentas.

Depois verificaria uso de CPU e event loop lag para detectar bloqueio por CPU-bound.
Em seguida observaria tempo de resposta de dependências (banco, cache, APIs externas), número de conexões e taxa de erro/timeout.

Também analisaria throughput, filas internas e saturação do pool de conexões.
O objetivo é identificar se o gargalo está na aplicação, no banco ou em um serviço externo antes de qualquer otimização.

---

#### 52) O que é event loop lag e por que ele é um dos sinais mais importantes de problema em uma aplicação Node.js?

Event loop lag é o atraso entre quando o Event Loop deveria executar a próxima tarefa e quando ele realmente consegue executá-la.

Ele indica que a thread principal está ocupada por operações bloqueantes, CPU-bound, GC intenso ou código síncrono pesado.

Se o lag aumenta, todas as requisições sofrem aumento de latência, mesmo que o banco esteja saudável.

É uma métrica crítica porque Node depende de um Event Loop livre para escalar; lag elevado, significa degradação global da aplicação.

---

#### 53) Resuma o princípio fundamental para escalar aplicações Node.js em produção.

O princípio fundamental para escalar aplicações Node.js é manter o Event Loop livre e delegar trabalho pesado ou demorado para fora da thread principal.

Isso signifca evitar operações bloqueantes, paralelisar I/O quando possível e desacoplar processamento pesado via fila ou workers.

Node escala bem quando a thread principal apenas orquestra I/O e responde rapidamente às requisições.

Quando CPU-bound, blocking I/O ou dependências lentas entram no request lifecycle, a escalabilidade colapsa.

> Escalar Node.js significa manter o Event Loop sempre livre, tratando a aplicação como orquestradora de I/O e delegando CPU-bound e trabalhos demorados para workers, filas ou serviços externos.