# Módulo 6 — Instruções Fundamentais (Parte 1)

> **Arquitetura:** x86-64
> **Sistema operacional de referência:** Linux
> **ABI:** System V AMD64
> **Sintaxe:** Intel

## 1. Introdução

Este módulo será dividido em várias partes, cada uma apresentando um pequeno grupo de instruções, com exercícios próprios. Isso é intencional: o objetivo não é decorar uma lista, mas assimilar cada instrução com calma antes de seguir para a próxima.

Nesta **Parte 1**, veremos: `mov`, `lea`, `push`, `pop`, `add`, `sub`.

## 2. Como vamos descrever cada instrução

Para cada instrução nova, vamos seguir sempre o mesmo formato:

- **O que faz** (descrição conceitual)
- **Sintaxe geral**
- **O que muda** (registradores, memória, flags)
- **Exemplo comentado**
- **Erro comum de interpretação**

## 2.1 Uma prévia necessária: o que é RFLAGS

Antes de começar a ver as instruções, vale explicar rapidamente um termo que vai aparecer repetidas vezes a partir daqui: **flags**.

**RFLAGS** é um registrador especial da CPU — assim como `RAX` ou `RBX`, mas com uma função bem diferente. Em vez de guardar um valor qualquer que você escolhe, cada **bit individual** de RFLAGS funciona como um interruptor (0 ou 1) que registra **uma característica do resultado da última operação executada**.

Por exemplo (vamos aprofundar cada uma no Módulo 7):

- **ZF** (*Zero Flag*): vira 1 se o resultado da última operação foi zero.
- **SF** (*Sign Flag*): vira 1 se o resultado da última operação foi negativo (bit mais significativo = 1, lembra do Módulo 4?).
- **CF** (*Carry Flag*): vira 1 se houve "estouro" em uma operação unsigned.
- **OF** (*Overflow Flag*): vira 1 se houve "estouro" em uma operação signed.

Um jeito simples de pensar nisso: depois de uma instrução como `sub eax, ebx`, a CPU não só calcula `eax - ebx` — ela também **anota metadados** sobre esse resultado (foi zero? foi negativo?) nos bits de RFLAGS. Essas anotações são o que permite, mais adiante, instruções de desvio condicional (`je`, `jl`, `jg`...) decidirem "para onde ir" com base no resultado de uma operação anterior.

Por ora, o que importa reter é só isto:

> **Nem toda instrução mexe em RFLAGS.** Algumas (como `mov`, `lea`, `push`, `pop`) deixam RFLAGS intocado. Outras (como `add`, `sub`) sempre atualizam essas flags como efeito colateral do cálculo.

Essa distinção — "afeta flags" vs. "não afeta flags" — é o que você vai ver marcado em cada instrução deste módulo, e é o motivo de existir uma coluna própria para isso na tabela-resumo (Seção 9). O funcionamento detalhado de cada flag fica para o Módulo 7.

## 3. `mov` — mover (copiar) um valor

### O que faz

Copia um valor de uma origem para um destino. **Não é uma "troca"** — o valor de origem permanece inalterado; apenas o destino é sobrescrito.

### Sintaxe geral

```asm
mov destino, origem
```

Em sintaxe Intel, o destino sempre vem **primeiro** (à esquerda), a origem **depois** (à direita). Isso é o oposto da sintaxe AT&T — vale reforçar, já que você pode encontrar exemplos nos dois formatos ao pesquisar por conta própria.

### O que muda

Apenas o destino. `mov` **não afeta flags** (RFLAGS permanece inalterado após um `mov`) — isso será relevante mais adiante, no Módulo 7.

### Exemplo comentado

```asm
mov eax, ebx        ; EAX recebe o valor de EBX. EBX não muda.
mov eax, 10          ; EAX recebe o valor imediato 10.
mov eax, [rbx]       ; EAX recebe o valor armazenado no endereço contido em RBX.
mov [rbx], eax       ; o endereço contido em RBX recebe o valor de EAX.
```

### Erro comum de interpretação

Achar que `mov` "move" no sentido de "remover da origem e colocar no destino" (como mover um arquivo entre pastas). Na verdade, é uma **cópia**: a origem continua com o mesmo valor depois da instrução.

## 4. `lea` — carregar endereço efetivo

### O que faz

`lea` (*Load Effective Address*) **calcula um endereço** a partir de uma expressão entre colchetes, mas **não acessa a memória** — ou seja, não faz dereferência. Ele apenas coloca o **resultado do cálculo do endereço** no destino.

Isso é uma das confusões mais comuns para quem está começando: `lea` **parece** com um acesso à memória (por causa dos colchetes), mas **não é**.

### Sintaxe geral

```asm
lea destino, [expressão de endereço]
```

### O que muda

Apenas o registrador de destino recebe o valor calculado (o endereço). Nenhuma leitura de memória ocorre.

### Exemplo comentado

```asm
lea eax, [rbx+8]     ; EAX = RBX + 8  (apenas o cálculo, sem ler memória)
mov eax, [rbx+8]     ; EAX = valor ARMAZENADO no endereço RBX+8 (aqui sim, há leitura)
```

Compare com o exemplo do Módulo 5 (Seção 5), sobre o operador `&` em C:

```c
int  a = 10;
int *p = &a;    // p recebe o ENDEREÇO de a — não o valor de a
```

`lea` é o equivalente conceitual, em Assembly, do operador `&` em C: ele calcula **onde** algo está, sem ler **o que** está lá.

### Uso comum: aritmética "disfarçada"

Como `lea` apenas calcula uma expressão do tipo `base + índice*escala + deslocamento`, compiladores frequentemente o usam como um **atalho para multiplicações e somas simples**, mesmo sem nenhuma relação com endereços de memória de verdade. Por exemplo:

```asm
lea eax, [rbx+rbx*2]   ; EAX = RBX + RBX*2 = RBX*3
```

Isso é um truque comum: calcular `RBX*3` em uma única instrução, aproveitando a capacidade de multiplicação por escala do endereçamento. Ao ler Assembly gerado por compiladores otimizados, é comum encontrar `lea` sendo usado apenas como calculadora, sem relação alguma com ponteiros.

### Erro comum de interpretação

Ler `lea eax, [rbx+8]` como "leia o valor no endereço RBX+8" (confundindo com `mov`). Lembre-se: **`lea` nunca acessa memória** — ele só calcula um número (o endereço) e o coloca no destino.

## 5. `push` — empilhar um valor

> **Nota rápida sobre RSP:** `RSP` (*Stack Pointer*) é um registrador de uso especial — diferente de `RAX`, `RBX` etc., que você usa livremente para qualquer valor, `RSP` é reservado pela convenção da arquitetura para **guardar o endereço do topo atual da stack**. A CPU e o sistema operacional esperam que `RSP` sempre aponte para essa região de memória; por isso, instruções como `push` e `pop` o modificam automaticamente, em vez de você precisar gerenciá-lo manualmente. Vamos explorar a stack como região de memória em profundidade no Módulo 9 — por ora, pense em `RSP` como "o marcador que diz onde é o topo da pilha agora".

### O que faz

Coloca um valor no topo da **stack** (pilha), uma região de memória que estudaremos em profundidade no Módulo 9. Por ora, o que importa: `push` decrementa o registrador `RSP` (Stack Pointer) e depois escreve o valor no novo topo.

### Sintaxe geral

```asm
push origem
```

### O que muda

- `RSP` diminui em 8 (em modo 64 bits, `push` sempre trabalha com blocos de 8 bytes/qword, mesmo que o valor "lógico" seja menor).
- O valor de `origem` é escrito no endereço de memória agora apontado por `RSP`.

### Exemplo comentado

```asm
push rax        ; RSP -= 8; o valor de RAX é escrito em [RSP]
```

### Erro comum de interpretação

Achar que `push` "cria espaço na memória do nada". Na verdade, a stack já é uma região de memória reservada para o programa desde o início da execução — `push` apenas move o ponteiro `RSP` dentro dessa região e escreve nela.

## 6. `pop` — desempilhar um valor

### O que faz

O oposto de `push`: lê o valor do topo atual da stack e o coloca no destino, depois incrementa `RSP`.

### Sintaxe geral

```asm
pop destino
```

### O que muda

- O valor no endereço apontado por `RSP` é lido e copiado para `destino`.
- `RSP` aumenta em 8.

### Exemplo comentado

```asm
pop rbx         ; RBX recebe o valor em [RSP]; RSP += 8
```

### Erro comum de interpretação

Achar que `pop` "apaga" o valor da memória. Na prática, o valor pode continuar fisicamente presente naquele endereço até ser sobrescrito por outra operação — `pop` apenas ajusta `RSP` para que aquele espaço seja considerado "livre" novamente, do ponto de vista da stack.

## 7. `add` — soma

### O que faz

Soma o valor de origem ao destino, e armazena o resultado no próprio destino.

### Sintaxe geral

```asm
add destino, origem
```

Equivale, conceitualmente, a `destino = destino + origem`.

### O que muda

- O valor de `destino` é sobrescrito pelo resultado da soma.
- **Flags são afetadas** (ZF, CF, SF, OF, entre outras) — veremos isso em detalhe no Módulo 7. Por ora, apenas registre que operações aritméticas, ao contrário de `mov`, alteram RFLAGS.

### Exemplo comentado

```asm
mov eax, 5
add eax, 3       ; EAX = EAX + 3 → EAX = 8
```

### Erro comum de interpretação

Achar que `add destino, origem` soma e guarda em um lugar separado, como em uma calculadora com "resultado" à parte. Não existe operando de resultado separado em `add` — o destino **é** onde o resultado é gravado, sobrescrevendo o valor anterior.

## 8. `sub` — subtração

### O que faz

Subtrai o valor de origem do destino, e armazena o resultado no próprio destino.

### Sintaxe geral

```asm
sub destino, origem
```

Equivale, conceitualmente, a `destino = destino - origem`.

### O que muda

Mesma lógica de `add`: o destino é sobrescrito, e flags são afetadas.

### Exemplo comentado

```asm
mov eax, 10
sub eax, 4       ; EAX = EAX - 4 → EAX = 6
```

### Erro comum de interpretação

Inverter a ordem mentalmente. `sub eax, ebx` é `eax = eax - ebx`, **não** `ebx - eax`. A ordem dos operandos importa, e o destino (esquerda) é sempre o "ponto de partida" da operação.

## 9. Tabela-resumo da Parte 1

| Instrução | Efeito | Acessa memória? | Afeta flags? |
|---|---|---|---|
| `mov d, o` | `d = o` | Depende dos operandos | Não |
| `lea d, [expr]` | `d = cálculo do endereço` | **Não**, apenas calcula | Não |
| `push o` | `RSP -= 8; [RSP] = o` | Sim (escreve na stack) | Não |
| `pop d` | `d = [RSP]; RSP += 8` | Sim (lê da stack) | Não |
| `add d, o` | `d = d + o` | Depende dos operandos | Sim |
| `sub d, o` | `d = d - o` | Depende dos operandos | Sim |

## 10. Exemplo prático integrado

```asm
mov rax, 10       ; RAX = 10
mov rbx, 3        ; RBX = 3
add rax, rbx      ; RAX = RAX + RBX → RAX = 13
push rax          ; RSP -= 8; memória[RSP] = 13
sub rax, 5        ; RAX = RAX - 5 → RAX = 8
pop rcx           ; RCX = memória[RSP] = 13; RSP += 8
```

Passo a passo do raciocínio:

1. `RAX` e `RBX` recebem valores imediatos.
2. `add` soma `RBX` a `RAX`, resultando em `RAX = 13`. `RBX` continua valendo 3.
3. `push rax` empilha o valor 13 (o valor atual de RAX naquele momento).
4. `sub rax, 5` altera `RAX` para 8. Note que isso **não afeta** o valor 13 que já foi empilhado — o `push` copiou o valor no momento em que foi executado.
5. `pop rcx` lê o valor do topo da stack (13, empilhado no passo 3) e o coloca em `RCX`. `RAX`, nesse momento, continua valendo 8 — os dois registradores são independentes.

Esse exemplo já ilustra uma ideia importante: **valores empilhados são cópias, congeladas no momento do `push`** — mudanças posteriores no registrador de origem não afetam o que já está na stack.

## 11. Exercícios

### Nível 1 — Interpretar uma instrução

1. O que faz `mov edx, 42`?
2. O que faz `lea eax, [rcx+4]`, e por que ela **não** é equivalente a `mov eax, [rcx+4]`?
3. O que acontece com `RSP` após um `push`? E após um `pop`?

### Nível 2 — Interpretar algumas instruções

Para o trecho abaixo, explique o efeito de cada linha:

```asm
mov eax, 20
mov ebx, 7
sub eax, ebx
add ebx, eax
```

4. Qual é o valor final de `EAX`?
5. Qual é o valor final de `EBX`?

### Nível 3 — Acompanhar registradores

```asm
mov rax, 100
mov rbx, 50
push rax
add rax, rbx
pop rcx
```

6. Qual é o valor de `RAX` ao final?
7. Qual é o valor de `RCX` ao final?
8. Explique por que `RCX` não recebe o valor final de `RAX`.

---

## 12. Respostas

1. Coloca o valor imediato 42 no registrador `EDX`, sobrescrevendo o que estava lá antes.
2. `lea eax, [rcx+4]` calcula o endereço `RCX + 4` e coloca esse **número** (o endereço em si) em `EAX` — sem acessar a memória. Já `mov eax, [rcx+4]` lê o **valor armazenado** no endereço `RCX + 4` e coloca esse valor em `EAX`. `lea` calcula um endereço; `mov` (com colchetes) acessa o conteúdo desse endereço.
3. Após um `push`, `RSP` **diminui** em 8 (a stack "cresce para baixo" em endereços de memória, algo que será aprofundado no Módulo 9). Após um `pop`, `RSP` **aumenta** em 8.
4. `mov eax, 20` → EAX = 20. `mov ebx, 7` → EBX = 7. `sub eax, ebx` → EAX = 20 - 7 = 13. `add ebx, eax` → EBX = 7 + 13 = 20. Valor final de `EAX`: **13**.
5. Valor final de `EBX`: **20** (calculado no passo anterior).
6. `mov rax, 100` → RAX = 100. `mov rbx, 50` → RBX = 50. `push rax` empilha o valor 100 (RAX continua 100). `add rax, rbx` → RAX = 100 + 50 = 150. `pop rcx` não afeta RAX. Valor final de `RAX`: **150**.
7. `pop rcx` lê o valor do topo da stack, que foi empilhado por `push rax` **antes** da soma — ou seja, o valor 100. Valor final de `RCX`: **100**.
8. Porque o `push rax` ocorreu **antes** da instrução `add rax, rbx`. O valor empilhado é uma cópia congelada de RAX no momento do `push` (100), e não é afetado por mudanças posteriores em RAX. Por isso `RCX`, ao receber o valor desempilhado, recebe 100 e não 150.

---

*Módulo anterior: [Módulo 5 — Memória](./05-memoria.md)*
*Próximo: [Módulo 6 — Instruções Fundamentais (Parte 2)](./06-instrucoes-fundamentais-parte2.md)*
