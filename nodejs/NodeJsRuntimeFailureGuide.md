# Node.js Runtime Failure Guide

Este documento descreve falhas clássicas de sistemas backend em Node.js em produção, por que acontecem e qual arquitetura resolve.

## Índice

- [1) Event Loop Blocking (CPU-Bound)](#1-event-loop-blocking-cpu-bound)
  - [Sintomas reais](#sintomas-reais)
  - [O que está acontecendo internamente](#o-que-esta-acontecendo-internamente)
  - [Por que isso acontece](#por-que-isso-acontece)
  - [Consequência operacional](#consequencia-operacional)
  - [Soluções](#solucoes)

---

## 1) Event Loop Blocking (CPU-Bound)

Uma API Node possui um endpoint que realiza uma operação pesada (criptografia, geração de relatório, parsing massivo, validação de schema grande compressão síncrona).
Durante carga simultânea moderada (ex: 50 usuários), toda a aplicação fica lenta e o healthcheck falha.

### Sintomas reais
- `/health` não responde
- todas as rotas ficam lentas
- apenas **um core** em 100%
- CPU total do servidor não está saturada
- logs não mostram erro

### O que está acontecendo internamente
O JavaScript do Node roda em  **um único call stack ativo**.

O event loop só consegue executar um callback por vez.

Quando uma função CPU-bound executa:

```
handler -> função pesada -> V8 ocupado -> event loop não roda
```

Enquanto isso:
- nenhuma requisição nova é processada
- promises não resolvem
- timers não executam
- conexões acumulam

Ou seja: não é apenas a rota pesada que para — **o servidor inteiro para**.

### Por que isso acontece

Node é eficiente quando está:
> esperando I/O

Ele é ineficiente quando:

> executa computação continua

Isso ocorre porque o V8 não é preemptivo.
Ele só libera controle quando a função retorna.

### Consequência operacional

O load balancer interpreta como instância não saudável.
Removendo instâncias -> concentra tráfego -> derruba cluster

### Soluções
- `worker_threads` para CPU pesada
- serviço separado de processamento
- fila + workers

Nunca:
- CPU pesado dentro do request HTTP