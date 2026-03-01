# Revisão de Problemas de Arquitetura

## Índice

- [API REST de pedidos (alto volume — 50k req/min)](#api-rest-de-pedidos-alto-volume-50k-reqmin)
- [Estoque concorrente (hot row no banco)](#estoque-concorrente-hot-row-no-banco)
- [Upload de vídeos grandes (800MB)](#upload-de-videos-grandes-800mb)
- [WebSockets (200k usuários online)](#websockets-200k-usuarios-online)
- [Webhooks externos](#webhooks-externos)
- [Reverse proxy vs Load balancer](#reverse-proxy-vs-load-balancer)
- [Webhooks + conexões simultâneas](#webhooks-conexoes-simultaneas)
- [O aprendizado central](#o-aprendizado-central)

---

## API REST de pedidos (alto volume — 50k req/min)

### Cenário
Marketplace chama `POST /orders` para criar pedidos.

Características:
- retires agressivos
- timeout do cliente ~2s
- requisições simultâneas
- operação envolve estoque + banco

### Como o Node falha

A API recebe mais requisições concorrentes do que o banco consegue processar

O código típico:

```
request -> controller -> transaction -> update stock -> commit
```

O que ocorre:
- muitas requests ficam aguardando conexão do pool
- Promises ficam pendentes
- sockets permanecem abertos

O Node não trava por CPU
Ele trava por **acúmulo de trabalho concorrente aguardando I/O externo**.

### Sintomas reais
- latência crescente
- p95 > 5s
- timeouts do cliente
- retries duplicados
- pedidos duplicados
- CPU baixa (20-40%)
- memória crescente

### Causa técnica
Node não cria thread por request.

Ele coordena tudo pelo event loop.
Quando muitas operações aguardam I/O:

```
Promises pendentes + sockets abertos + callbacks registrados
```

o runtime fica congestionado

Isso é chamado:
**event loop saturation by pending I/O**

### Solução correta

#### Admission control + idempotência
**1.** limitar concorrência de acesso ao banco (semaphore)
**2.** responder 503 rápido quando saturado
**3.** idempotency key

Fluxo:
```
request
-> tenta adquirir slot
-> não conseguiu -> 503 Retry-After
-> conseguiu -> executa
```

Resultado:
- sem timeout
- cliente faz retry saudável
- sem duplicação

## Estoque concorrente (hot row no banco)

### Cenário
Muitos clientes compram o mesmo produto ao mesmo tmepo.

Tabela:
```
product(id, stock)
```

### Como o Node falha

Todos executam:

```sql
UPDATE product SET stock = stock - 1 WHERE id=1;
```

O banco cria:

```
fila de locks

```

Enquanto espera:

Node fica:

```
await query
```

e segura:
- conexão
- memória
- slot de concorrência

### Sintomas

- latência gigante
- throughput despenca
- retries aumentam
- sistema parece "lento" sem CPU alta

### Causa técnica

Row-level locking -> transações esperando lock.

Isso gera:

**lock amplification -> connection retention -> Node saturation**

### Solução

#### Reservation Pattern + SKIP LOCKED

Criar unidades de estoque:

```
stock_items(id, product_id, status)
```

Reserva:

```sql
SELECT id
FROM stock_items
WHERE product_id=1 AND status='AVAILABLE'
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

Cada compra pega uma linha diferente.

Sem fila de lock.

### Conceitos
- hot row
- lock contention
- transactional modeling
- reservation system
- convergent state

## Upload de vídeos grandes (800MB)

### Cenário

Usuários fazem upload lento (~10 minutos)

### Como o Node falha

Implementação ingênua:

```javascript
req.on('data', chunk => buffer.push(chunk))
```
ou
```javascript
Buffer.concat(...)
```

### Sintomas
- OOM kill
- GC 100%
- servidor reinicia
- poucos usuários derrubam sistema

### Causa técnica
O Node bufferiza dados quando não há backpressure.

800MB x uploads simultâneos = RAM esgotada.

### Solução

#### Streaming pipeline + S3
Nunca armazenas o arquivo no Node.

```
req (Readable stream)
-> pipeline
-> S3 upload stream
```

Node vira apenas um proxy de bytes.

### Problema adicional
Mesmo com streaming:

5.000 uploads simultâneos -> **file descriptor exhaustion (EMFILE)**

Cada upload mantém sockets abertos.

### Solução real

#### Pre-signed upload

```
cliente -> S3 direto
Node apenas autoriza
```

### Conceitos
- streams
- backpressure
- highWaterMark
- file descriptors
- slow clients

## WebSockets (200k usuários online)

### Cenário
Sistema de mensagens em tempo real.

### Como o Node falha
Cada usuário

```
1 socket aberto permanente
```

Node armazena cada conexão no heap

### Sintomas
- mensagens atrasadas
- disconnects aleatórios
- reconnect storm
- CPU baixa
- GC alto

### Causa técnica
Não é CPU

É:

**pressão de garbage collector**
Cada conexão é um objeto JS.
O GC precisa varrer dezenas de milhares de sockets.

Event loop pausa -> clients reconectam -> avalanche.

### Solução

Arquitetura:

```
WebSocket Gateway (leve)
+ Redis/Kafka pubsub
+ limite de conexões por instância
+ autoscaling por conexões
```

Opcional:
- usar serviço gerenciado
- separar realtime da API

### Conceitos
- connection state
- GC pauses
- fanout
- pub/sub routing
- reconnection storm

## Webhooks externos

### Cenário
Gateway enviam eventos
- repetidos
- fora de ordem
- simultâneos

### Como o Node falha
Processar webhook no controller:

```
HTTP -> lógica -> banco -> APIs externas
```

Provoca:
- timeout do provedor
- retries massivos
- duplicações

### Sintomas
- eventos duplicados
- estado inconsistente
- tráfego explosivo

### Causa técnica

Webhook não é request

É:

**ingestão de mensagem não confiável**

### Solução

#### Inbox Pattern

```
recebe -> grava no banco -> responde 200
worker processa depois
```

Com:

- idempotência por event_id
- máquina de estados monotônica

### Outro problema Node específico

Rajada de 5.000 webhooks simultâneos

**accept queue / SYN backlog overflow**
O kernel descarta conexões antes do Node aceitar.

### Mitigação
- reverse proxy (NGINX)
- cluster
- resposta imediata

### Conceitos
- idempotency
- eventual consistency
- accept backlog
- retry amplification

## Reverse proxy vs Load balancer

### Problema
Node recebe tráfego direto da internet

Falha em:
- burts
- TLS handshake
- slow clients

### Solução

Arquitetura correta

```
Internet
-> Load Balancer (distribui)
-> Reverse Proxy (amortece)
-> Node
```

Funções:

**Load balancer**
- escolhe instância

**Reverse proxy**
- estabiliza conexões
- buffering
- proteção ao event loop

### Conceitos
- edge server
- connection burst buffering
- keep-alive handling

## Webhooks + conexões simultâneas

Problema específico do Node:

**accept queue saturation**
O Node aceita conexões em um único loop.

Muitas conexões simultâneas:

```
kernel derruba conexões antes do JS rodar
```

Proxy resolve pois é otimizado em C para aceitar conexões massivas.

## O aprendizado central

Node raramente falha por CPU

Ele falha por
- coordenação de concorrência
- retenção de recursos
- tempo de vida das operações

Ou resumindo:
> Node não quebra quando trabalha demais
> Node quebra quando espera demais ao mesmo tempo.

