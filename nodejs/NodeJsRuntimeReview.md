# Node.js Runtime Survival Guide

## Índice

- [O princípio central](#o-principio-central)
- [O recurso mais crítico do Node](#o-recurso-mais-critico-do-node)
- [Os 4 tipos clássicos de queda](#os-4-tipos-classicos-de-queda)
  - [1) Event Loop Blocking (CPU-bound)](#1-event-loop-blocking-cpu-bound)
  - [2) Blocking I/O (Sync APIs)](#2-blocking-io-sync-apis)
  - [3) Thread Pool Saturation](#3-thread-pool-saturation)
  - [4) Memory Pressure / GC Collapse](#4-memory-pressure-gc-collapse)
- [Como o Garbage Collector te derruba](#como-o-garbage-collector-te-derruba)
- [Streams vs Buffer (o ponto mais importante)](#streams-vs-buffer-o-ponto-mais-importante)
- [Upload correto](#upload-correto)
- [Webhooks (muito importante)](#webhooks-muito-importante)
- [Filas — o verdadeiro motivo de existirem](#filas-o-verdadeiro-motivo-de-existirem)
- [Idempotência](#idempotencia)
- [Regra final para backend Node](#regra-final-para-backend-node)

---

<a id="o-principio-central"></a>
## O princípio central

> Node não é ruim com carga.
> Node é ruim quando você segura recursos dentro do processo.

Ele foi projetado para **orquestrar I/O**, não para **reter trabalho**.


<a id="o-recurso-mais-critico-do-node"></a>
## O recurso mais crítico do Node

Não é CPU
Não é RAM

É:
**tempo de posse do event loop**

Enquanto seu código executa no thread principal:

  - nenhuma outra requisição roda
  - nenhum timer roda
  - nenhum healthcheck responde

<a id="os-4-tipos-classicos-de-queda"></a>
## Os 4 tipos clássicos de queda

<a id="1-event-loop-blocking-cpu-bound"></a>
### 1) Event Loop Blocking (CPU-bound)

#### Causa
- loops grandes
- JSON.parse gigante
- criptografia síncrona
- compressão síncrona

#### Sintoma
- servidor congela
- todas as rotas param
- `/health` não responde
- CPU alto em 1 core

#### Por quê
O JavaScript roda em um único thread.
Código síncrono monopoliza o V8.

#### Solução
- worker_threads
- fila de jobs
- nunca CPU pesado no request HTTP

<a id="2-blocking-io-sync-apis"></a>
### 2) Blocking I//O (Sync APIs)

#### Causa
- `fs.readFileSync`
- `execSync`
- `zlibSync`

#### Sintoma
- servidor para mesmo com CPU baixa
- latência gigante
- timeouts

#### Por quê
O thread principal fica esperando o kernel.

#### Solução
Sempre usar APIs assíncronas

<a id="3-thread-pool-saturation"></a>
### 3) Thread Pool Saturation

#### Causa
- bcrypt/pbkdf2
- crypto pesado
- compressão massiva
- muitos fs async simultâneos

#### Sintoma
- CPU não tão alta
- servidor extremamento lento
- p95 enorme

#### Por quê
O libuv tem apenas **4 threads** por padrão

#### Consequência perigosa
Endpoint de login pode derrubar toda a API.

#### Soluções
- rate limit
- workers separados
- fila
- não expor computação pesada ao público

<a id="4-memory-pressure-gc-collapse"></a>
### 4) Memory Pressure / GC Collapse

#### Causa
- uploads em buffer
- payloads grandes guardados
- cache local sem limite
- muitos objetos vivos simultaneamente

#### Sintoma
- funciona por horas
- cai no pico
- GC rodando sempre
- depois: *heap out of memory*

#### Por quê
O V8 é otimizado para objetos curtos, não grandes.

O problema não é alocar
O problema é **reter**.

<a id="como-o-garbage-collector-te-derruba"></a>
## Como o Garbage Collector te derruba
O V8 tem duas áreas:

| Área             | Função                      |
| ---------------- | --------------------------- |
| Young Generation | objetos curtos (rápido)     |
| Old Generation   | objetos persistentes (caro) |

Quando muitos objetos sobrevivem:

-> vão para Old Space
-> Major GC
-> **pausa global do event loop**

Isso causa:
timeouts -> retries -> avalanche -> queda

<a id="streams-vs-buffer-o-ponto-mais-importante"></a>
## Streams vs Buffer (o ponto mais importante)

#### Buffer
- carrega tudo
- memória proporcional ao tamanho
- simples
- perigoso sob concorrência

#### Stream
- trafega dados
- memória constante
- backpressure automático
- seguro

**Regra prática**
Use **buffer** quando:
- < 500KB
- vida curta
- baixa concorrência

Use **stream quando**:
- arquivos
- upload
- download
- S3
- dados grandes
- tamanho desconhecido

### A regra de ouro
> Node escala quando move dados
> Node cai quando possui dados

<a id="upload-correto"></a>
## Upload correto

ERRADO:
```
recebe -> guarda -> processa -> envia
```

CERTO:

```
recebe -> processa em fluxo -> encaminha
```

Nunca:
- `Buffer.concat` em upload
- `multer.memoryStorage` para arquivo grande
- `sharp().toBuffer()` em produção
- enviar Buffer grande ao S3

<a id="webhooks-muito-importante"></a>
## Webhooks (muito importante)
Nunca processe webhook dentro do request.

Fluxo correto:
```
recebe -> valida mínimo -> persiste -> 200 OK -> processa depois
```

Por quê:
- provedores fazem retry agressivo
- resposta lenta cria avalanche de tráfego

<a id="filas-o-verdadeiro-motivo-de-existirem"></a>
## Filas — o verdadeiro motivo de existirem
Filas não servem para performance

Servem para:
- absorver picos
- desacoplar sistemas
- proteger o event loop
- garantir durabilidade

<a id="idempotencia"></a>
## Idempotência
Filas garantem **at-least-once**

Então você DEVE garantir:

```
processar duas vezes === mesmo resultado
```

Normalmente com:

- chave única no banco
- update condicional
- estado imutável

<a id="regra-final-para-backend-node"></a>
## Regra final para backend Node
Sempre pergunte:

> "Isso vai manter dados grandes vivos dentro do processo?"

Se sim:
- não é problema de código
- é problema de arquitetura