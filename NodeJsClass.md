# Resumo — Node.js Runtime, Event Loop, Async/Await e Closures

Este documento consolida os principais conceitos discutidos: arquitetura do Node.js, funcionamento do Event Loop, fluxo de execução assíncrono, microtasks, workers e closures.

---

## 1) Arquitetura do Node.js (quem é quem)

```
Seu código JS
      ↓
Node.js Runtime
    ├── V8 (motor JavaScript)
    │     ├── Call Stack
    │     ├── Heap (objetos e closures)
    │     └── Microtask Queue (Promise/await)
    │
    ├── Node Core (bindings C++)
    │     ├── fs, http, timers, process, Buffer
    │     └── ponte entre JS e sistema operacional
    │
    └── libuv
          ├── Event Loop
          ├── Thread Pool
          └── Integração com kernel (epoll/kqueue/IOCP)
```

### Papéis

- **V8:** executa JavaScript.
- **Node Core (bindings C++):** traduz chamadas JS para chamadas nativas.
- **libuv:** coordena IO assíncrono e eventos do sistema operacional.

**O Node Core conecta as APIs JavaScript ao sistema nativo (via libuv ou outras bibliotecas) e registra callbacks no event loop; quando o loop detecta que um evento terminou, o Node Core pede ao V8 para executar o callback na call stack.**

---

## 2) Call Stack vs Event Loop

### Call Stack

- Onde funções JavaScript executam.
- Só uma função JS executa por vez (single‑threaded).

### Event Loop

- Não executa código JS diretamente.
- Decide quando pedir ao V8 para executar um callback.
- Recebe notificações do sistema operacional.

> O event loop é o despachante. A call stack é o executor.

---

## 3) Fases do Event Loop (simplificado)

**1.** timers (setTimeout, setInterval) <br>
**2.** pending callbacks <br>
**3.** idle/prepare <br>
**4.** poll (I/O concluído — sockets, fs, etc.) <br>
**5.** check (setImmediate) <br>
**6.** close callbacks <br>

Após **cada callback JS**:

```
Call stack esvazia
→ process.nextTick
→ Promises (microtasks)
→ volta ao event loop
```

---

## 4) Microtasks vs Macrotasks

| Tipo      | Onde roda | Exemplos              |
| --------- | --------- | --------------------- |
| Microtask | V8        | Promise.then, await   |
| Macrotask | libuv     | setTimeout, I/O, HTTP |

Regra:

> Microtasks sempre executam antes do próximo evento do event loop.

---

## 5. Sockets, Kernel e como o servidor HTTP realmente funciona

Quando você faz:

```js
http.createServer(handler).listen(3000);
```

não é o Node que "fica verificando conexões". O que acontece:

```
JS chama listen
→ binding C++
→ libuv
→ kernel abre um socket TCP na porta 3000
```

O **kernel** passa a escutar a porta e manter uma fila de conexões pendentes (accept queue).

A libuv registra esse socket no mecanismo de notificação do SO:

- **Linux**: epoll
- **macOS**: kqueue
- **Windows**: IOCP

O event loop então fica bloqueado aguardando eventos:

```
epoll_wait / kqueue / IOCP
```

Ou seja, CPU praticamente 0% enquanto o servidor está ocioso.

### Quando um cliente conecta

**1.** Cliente abre TCP (SYN) <br>
**2.** Kernel aceita a conexão <br>
**3.** Kernel acorda a libuv <br>
**4.** Event loop entra na fase poll <br>
**5.** Node cria req e res <br>
**6.** Node chama o handler no V8 (call stack) <br>

Fluxo:

```
Kernel detecta conexão
→ libuv recebe evento de legibilidade
→ Node faz accept()
→ MakeCallback no V8
→ handler(req, res) executa
```

### Depois da resposta

Quando `res.end()` termina:

```
call stack esvazia
→ microtasks
→ event loop volta a dormir aguardando kernel
```

> O servidor HTTP "vive" no kernel (socket aberto). O Node apenas reage aos eventos desse socket.

---

## 6. Fluxo real de uma requisição HTTP

```
Cliente conecta
→ Kernel aceita conexão
→ libuv detecta evento
→ Node cria req/res
→ V8 executa handler (call stack)
→ handler termina
→ microtasks
→ event loop continua
```

O servidor "ocioso" está apenas bloqueado aguardando eventos do sistema operacional.

---

## 7. Async/Await (o que realmente acontece)

Código:

```js
const user = await db.query();
```

Internamente:

```
await
→ função pausa
→ retorna Promise pendente
→ call stack fica livre
→ outra requisição pode executar
→ IO termina
→ Promise resolve
→ microtask agenda continuação
→ função continua depois do await
```

> O Node não executa requisições em paralelo; ele intercala execução enquanto espera IO.

---

## 8. Thread Pool vs Worker Threads

### Thread Pool (libuv)

- Executa código C/C++ bloqueante.
- Não executa JavaScript.
- Usado por: fs, crypto, zlib, dns.lookup.

### Worker Threads (`new Worker()`)

- Outra instância completa do V8.
- Possui:
  - nova call stack
  - novo heap
  - novo event loop
- Executa JavaScript em paralelo real.

Comunicação:

```
postMessage ↔ fila interna ↔ outro event loop
```

Não usa socket.

---

## 9. O que é Closure

### Definição:

> Closure é a combinação de uma função com o ambiente léxico onde ela foi criada.

Ou seja:

```
função + variáveis externas preservadas
```

Exemplo:

```js
function criar() {
  let x = 10;
  return () => console.log(x);
}
```

Mesmo após `criar()` terminar, `x` continua existindo.

Por quê?

- O V8 move x da stack para a heap.
- A função guarda um ponteiro para esse ambiente.

---

## 10. Closure e Garbage Collector

O GC usa **alcançabilidade (reachability)**:

```
se existe referência → não coleta
se não existe → remove
```

Closure mantém dados vivos enquanto a função for acessível:

```
variável global → função → closure → variável
```

Quando a referência some:

```
f = null → GC remove tudo
```

### Risco comum

Handlers, timers e listeners podem manter objetos grandes vivos → memory leak.

---

## 11. O clássico: var vs let no loop

Código:

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

Saída:

```
3 3 3
```

Porque:

- Existe uma única variável i
- Closure captura a variável, não o valor

### Com `let`

```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

Saída:

```
0 1 2
```

O motivo:

> O for com let cria um novo ambiente léxico por iteração (per‑iteration environment).

Cada callback fecha sobre uma variável diferente.

---

# Bônus

## Frases-chave para entrevistas

O event loop não executa JS; ele agenda execução no V8.
O Node é single‑threaded para JavaScript, mas concorrente para IO.
await pausa a função, não a thread.

Microtasks executam antes da próxima fase do event loop.

Closure captura variável (referência), não valor.

let cria novo ambiente por iteração; var compartilha o mesmo.

O GC coleta por alcançabilidade, não por escopo.

## Modelo mental final

Sistema operacional gera eventos
→ libuv detecta
→ Node traduz
→ V8 executa
→ microtasks
→ event loop continua

E closures permitem que o JavaScript lembre de dados mesmo depois que a função original já terminou.

---
