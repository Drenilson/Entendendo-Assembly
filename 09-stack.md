# Módulo 9 — Stack

> **Arquitetura:** x86-64
> **Sistema operacional de referência:** Linux
> **ABI:** System V AMD64
> **Sintaxe:** Intel

## 1. Introdução

Desde o Módulo 6, `RSP` (*Stack Pointer*) já aparecia em `push` e `pop`, sempre com a promessa de que a stack seria detalhada mais adiante. Também já vimos, no Módulo 8 (Seção 1.1), uma forma simbólica de acessar variáveis (`[i]`, `[soma]`), com a promessa de que a forma real, baseada em deslocamentos de um registrador, seria explicada aqui.

Este módulo cumpre as duas promessas. O objetivo é entender:

- o que a stack realmente é, como região de memória;
- por que existe um segundo registrador, `RBP`, além de `RSP`;
- como uma função organiza suas variáveis locais e seus argumentos dentro dessa região;
- e, com isso, conseguir olhar para uma função Assembly completa e identificar suas três partes: **entrada, processamento e saída**.

A convenção completa de chamada de função (como argumentos são passados, quais registradores são preservados) fica para o Módulo 10. Aqui, o foco é a stack em si, como estrutura de memória que sustenta tudo isso.

## 2. Revisão rápida: `RSP` e `push`/`pop`

Do Módulo 6 (Parte 1): `push` decrementa `RSP` em 8 e escreve um valor no novo topo; `pop` lê o valor do topo e incrementa `RSP` em 8. Isso já é, na prática, a mecânica básica da stack. O que falta é entender **por que** ela é organizada dessa forma, e como isso se conecta a funções inteiras, não apenas a instruções isoladas.

## 3. A stack como região de memória

A stack é uma região de memória RAM reservada para o programa, assim como qualquer outra. O que a torna especial não é o hardware, mas a **convenção de uso**: ela funciona como uma pilha no sentido literal, o último valor colocado é o primeiro a ser retirado (*LIFO*, *Last In, First Out*).

Esta seção existe para resolver, com números concretos, uma confusão extremamente comum: a ideia de que "a stack cresce para baixo" entra em conflito com a ideia de "empilhar é colocar coisas de baixo para cima". Essas duas frases parecem se contradizer, mas na verdade descrevem coisas diferentes: uma fala do **valor numérico do endereço**; a outra, de uma **imagem mental de empilhar objetos físicos**. Vamos separar as duas com cuidado.

### 3.1 Acompanhando endereços reais, passo a passo

Esqueça, por um momento, qualquer desenho ou imagem. Vamos só seguir números. Suponha que, antes de qualquer `push`, `RSP` valha `0x2000`.

```asm
mov rsp, 0x2000    ; estado inicial, hipotético
push rax           ; suponha RAX = 5
push rbx           ; suponha RBX = 10
```

Aplicando exatamente a mecânica do Módulo 6 (`push` decrementa `RSP` em 8, depois escreve no novo topo):

| Ação | Novo valor de `RSP` | O que foi escrito, e onde |
|---|---|---|
| Antes de qualquer push | `0x2000` | (nada empilhado ainda; `RSP` aqui é só o "teto" da região) |
| Após `push rax` | `0x1FF8` | `5` é escrito no endereço `0x1FF8` |
| Após `push rbx` | `0x1FF0` | `10` é escrito no endereço `0x1FF0` |

Repare no padrão, olhando só para a coluna de endereços: `0x2000` → `0x1FF8` → `0x1FF0`. **Cada novo valor empilhado ocupa um endereço menor que o anterior.** Não é "endereço baixo para endereço alto"; é o oposto. O valor mais recentemente empilhado (`10`, em `0x1FF0`) está no endereço **mais baixo** entre os três, e é justamente ele o "topo da pilha" agora, porque é ele quem será removido primeiro por um eventual `pop`.

Fazendo o caminho de volta:

```asm
pop rcx   ; RCX = valor em [RSP] = 10 (lido de 0x1FF0); RSP volta a 0x1FF8
```

Depois desse `pop`, `RSP` volta a `0x1FF8`, que agora é novamente o topo, e contém o valor `5` (empilhado antes).

> **A confusão, resolvida:** "empilhar" (no sentido de LIFO, o último a entrar é o primeiro a sair) não tem relação direta com o endereço aumentar ou diminuir. O que importa é a **ordem de remoção**: o valor mais recente é sempre o mais fácil de acessar (o topo), e em x86-64 isso corresponde ao **menor endereço** entre os valores atualmente empilhados. "Crescer para baixo" descreve exatamente essa mecânica: a cada novo item, o endereço do topo diminui.

### 3.2 Por que isso parece contraintuitivo

A origem da confusão é bem específica: quando pensamos em "empilhar" no sentido físico (uma pilha de pratos, uma pilha de livros), imaginamos cada novo item sendo colocado **acima** do anterior, crescendo para cima no espaço físico. Essa intuição não está errada em si, ela só não tem nenhuma relação obrigatória com "o valor do endereço aumentar". "Acima" e "abaixo", numa pilha física, descrevem posição no espaço; "maior" e "menor", em endereços de memória, descrevem apenas um número. São duas escalas diferentes, e o erro comum é tentar encaixar as duas na mesma direção sem verificar.

Uma vez que fixamos os números (Seção 3.1) como a fonte de verdade, qualquer desenho pode ser construído em cima deles, sem ambiguidade. É isso que a próxima seção faz.

### 3.3 Um único desenho, usado de forma consistente

Para evitar ambiguidade, este curso vai usar sempre a mesma convenção de desenho: **endereços mais altos ficam no topo do diagrama; endereços mais baixos ficam embaixo** (é a forma mais comum em material de sistemas operacionais e depuradores). Usando os números da Seção 3.1:

```
0x2000  ┌────────────────────────── ← endereço mais alto (região "livre", ainda não usada pela stack)
        │  (memória não usada ainda)   │
0x1FF8  ├──────────────────────────
        │   5   (empilhado por RAX)    │
0x1FF0  ├──────────────────────────
        │  10   (empilhado por RBX)    │
        └────────────────────────── ← RSP aponta aqui (topo atual, após os dois pushes)
                       │
                       ▼
              endereços mais baixos
```

Nesta convenção (a que vamos manter em todo o curso), a stack cresce **descendo visualmente no desenho**, na mesma direção em que os números diminuem. Não há contradição: o novo valor empilhado (`10`) está fisicamente mais abaixo no desenho **e** em um endereço numericamente menor, ao mesmo tempo. A ideia de "empilhar de baixo para cima" só voltaria a fazer sentido visual se, em vez disso, desenhássemos os endereços baixos no topo, o que é uma escolha de desenho válida em outros contextos (alguns depuradores fazem isso), mas que este curso não vai usar, exatamente para evitar essa ambiguidade.

> **Regra fixa deste curso:** sempre que houver um diagrama de memória, endereço mais alto fica em cima, endereço mais baixo fica embaixo. "A stack cresce para baixo" pode, a partir de agora, ser lido tanto como "os números diminuem" quanto como "desce no desenho", porque as duas leituras sempre vão coincidir aqui.

### 3.4 Onde a stack fica dentro do espaço de endereços de um processo Linux

Para dar um contexto mais concreto (sem entrar em detalhes que fogem do escopo deste curso), vale saber, em linhas gerais, como um processo em Linux x86-64 organiza sua memória. Usando a mesma convenção da Seção 3.3 (alto em cima, baixo embaixo):

```
Endereços mais altos
        ┌──────────────────────────
        │      (espaço do kernel)      │
        ├──────────────────────────
        │           ↓ stack ↓          │  ← cresce para baixo (Seção 3.1)
        ├──────────────────────────
        │     (região não mapeada)     │
        ├──────────────────────────
        │           ↑ heap  ↑          │  ← cresce para cima (endereços aumentando)
        ├──────────────────────────
        │       dados do programa      │
        ├──────────────────────────
        │      código do programa      │
        └──────────────────────────
Endereços mais baixos
```

Note que a **stack** e a **heap** (a região usada por alocação dinâmica, como `malloc` em C, fora do escopo deste curso por ora) crescem em direções opostas, uma vindo de cima para baixo, a outra de baixo para cima, uma em direção à outra. Isso não é acidental: é uma forma eficiente de aproveitar a região de memória disponível entre as duas, sem precisar decidir de antemão quanto espaço cada uma vai efetivamente usar.

## 4. `RBP` — Base Pointer, e por que ele existe

`RSP` muda o tempo todo durante a execução de uma função: cada `push`, `pop`, chamada de outra função ou alocação temporária de espaço desloca `RSP`. Isso o torna um péssimo ponto de referência fixo para acessar variáveis locais e argumentos: se uma variável estivesse marcada como "8 bytes abaixo de `RSP`", essa distância mudaria a cada `push` ou `pop` executado dentro da função, exigindo recalcular tudo constantemente.

A solução adotada pela convenção: no início de cada função, um segundo registrador, `RBP` (*Base Pointer*), recebe uma cópia do valor de `RSP` **naquele momento específico**, e permanece **fixo** durante toda a execução da função (até o momento de retornar). A partir daí, todas as variáveis locais e argumentos são referenciados como deslocamentos fixos a partir de `RBP`, algo como `[rbp-4]` ou `[rbp+16]`, que nunca mudam durante a execução da função, mesmo que `RSP` continue se movendo por conta de operações internas.

> **RBP é a "âncora" da função; RSP é o "ponteiro móvel do topo".** Essa distinção de papéis é o que motiva a existência dos dois registradores separadamente, mesmo os dois lidando com a mesma região de memória.

## 5. A região de uma função na stack: o *stack frame*

O trecho da stack usado por uma única chamada de função, delimitado (na prática) entre o valor de `RBP` e o valor atual de `RSP`, é chamado de **stack frame** (ou "quadro de pilha"). Cada vez que uma função é chamada, um novo stack frame é criado; quando ela retorna, esse frame é desfeito, e o frame da função que a chamou volta a ser o "atual".

Isso já antecipa algo importante para o Módulo 10: como funções podem chamar outras funções, a stack acumula um frame por cada chamada ainda não retornada, empilhados uns sobre os outros, exatamente como o nome sugere.

## 6. Prólogo e epílogo: o padrão de início e fim de uma função

Praticamente toda função compilada segue um padrão fixo no início (**prólogo**) e no fim (**epílogo**), responsável por configurar e desfazer o stack frame.

### 6.1 Prólogo

```asm
push rbp         ; salva o RBP da função anterior (quem chamou esta função)
mov rbp, rsp     ; RBP passa a apontar para o topo atual: início do novo frame
sub rsp, N       ; reserva N bytes de espaço na stack para variáveis locais
```

Passo a passo:

1. `push rbp`: antes de sobrescrever `RBP`, seu valor atual (pertencente à função anterior, a que chamou esta) é salvo na stack, para poder ser restaurado depois.
2. `mov rbp, rsp`: `RBP` recebe o valor atual de `RSP`, esse é o ponto de partida fixo para todas as referências dentro desta função.
3. `sub rsp, N`: `RSP` é deslocado para baixo em `N` bytes, reservando espaço na stack para as variáveis locais da função. Esse espaço fica "entre" o novo `RBP` e o novo `RSP`.

### 6.2 Epílogo

```asm
mov rsp, rbp      ; descarta o espaço reservado para variáveis locais
pop rbp           ; restaura o RBP da função anterior
ret               ; retorna para quem chamou (Módulo 10 detalha essa instrução)
```

Passo a passo:

1. `mov rsp, rbp`: `RSP` volta a apontar para onde `RBP` está, desfazendo o espaço reservado no prólogo (isso é equivalente ao efeito de vários `pop`s, ou de somar de volta o `N` que foi subtraído).
2. `pop rbp`: o valor de `RBP` salvo no início (pertencente à função chamadora) é restaurado, devolvendo `RBP` ao estado de antes desta função ser chamada.
3. `ret`: a execução retorna para o ponto de onde esta função foi chamada.

> **Atalho comum:** muitos compiladores usam a instrução `leave` para substituir as duas primeiras linhas do epílogo (`mov rsp, rbp` seguido de `pop rbp`) por uma única instrução equivalente. Ao ler `leave`, basta lembrar que ela faz exatamente essas duas coisas.

### 6.3 Visualizando o efeito do prólogo na stack

Vamos aplicar o mesmo estilo de acompanhamento numérico da Seção 3.1, agora ao prólogo. Suponha que, no momento em que esta função é chamada, `RSP` valha `0x3000`, e que o `RBP` da função anterior valha `0x5000` (um valor qualquer, de outro ponto da stack).

| Instrução executada | `RSP` depois | `RBP` depois | O que existe no endereço mais recente |
|---|---|---|---|
| (antes do prólogo) | `0x3000` | `0x5000` (da função anterior) | (nada ainda) |
| `push rbp` | `0x2FF8` | `0x5000` (ainda não mudou) | `0x5000` é escrito em `0x2FF8` (o RBP antigo, salvo) |
| `mov rbp, rsp` | `0x2FF8` | `0x2FF8` (copiado de RSP) | (nada escrito, só uma cópia entre registradores) |
| `sub rsp, 16` | `0x2FE8` | `0x2FF8` (continua fixo) | (espaço reservado, ainda sem conteúdo definido) |

Usando a mesma convenção de desenho da Seção 3.3 (endereço mais alto em cima):

```
0x3000  ┌──────────────────────────  ← RSP estava aqui, antes do prólogo
        │   (memória de outra função)  │
0x2FF8  ├──────────────────────────  ← RBP aponta aqui (RBP antigo, 0x5000, foi salvo neste endereço)
        │   0x5000  (RBP antigo)       │
0x2FE8  ├────────────────────────── 
        │   espaço reservado (16       │
        │   bytes) para variáveis      │
        │   locais desta função        │
        └──────────────────────────  ← RSP aponta aqui, após "sub rsp, 16"
```

Repare que `RBP` ficou parado em `0x2FF8` assim que foi definido, e vai continuar ali durante toda a função, mesmo que `RSP` continue se movendo (por exemplo, se a função fizer mais `push`s internamente). É exatamente essa estabilidade que permite usar `[rbp-4]`, `[rbp-8]` como referências fixas para as variáveis locais reservadas nesses 16 bytes.

## 7. Variáveis locais: deslocamentos negativos a partir de `RBP`

Uma vez que `RBP` está fixo e aponta para o início do frame, variáveis locais (que ficam no espaço reservado por `sub rsp, N`) são acessadas com deslocamentos **negativos**, contando "para baixo" a partir de `RBP`.

```asm
mov dword [rbp-4], 10    ; primeira variável local, 4 bytes abaixo de RBP
mov dword [rbp-8], 20    ; segunda variável local, 8 bytes abaixo de RBP
```

Essa é exatamente a forma "real" que ficou pendente do Módulo 8: onde antes usávamos `[i]` como apelido legível, agora sabemos que, na prática, isso corresponde a algo como `[rbp-4]`, um deslocamento fixo dentro do stack frame da função atual.

## 8. Argumentos: deslocamentos positivos a partir de `RBP` (uma prévia)

Diferente das variáveis locais, os **argumentos** de uma função (quando não ficam apenas em registradores, algo que o Módulo 10 vai detalhar) costumam ser acessados com deslocamentos **positivos** a partir de `RBP`, porque eles foram empilhados **antes** do prólogo desta função ser executado, ou seja, estão em endereços "acima" de `RBP` (lembrando que a stack cresce para baixo, então "antes" corresponde a endereços mais altos).

```asm
mov eax, [rbp+16]   ; um argumento, acima do RBP atual
```

> Este módulo apresenta apenas a mecânica geral de deslocamentos positivos para argumentos. A System V AMD64 ABI, na prática, prioriza passar os primeiros argumentos diretamente por registradores (`RDI`, `RSI`, `RDX`...), reservando a stack principalmente para os argumentos que excedem essa quantidade, ou para casos específicos. Essa convenção completa, incluindo quais registradores correspondem a qual posição de argumento, é o assunto central do Módulo 10.

## 9. Exemplo integrado: a função `soma`

Vamos aplicar tudo isso a uma função C simples:

```c
int soma(int a, int b) {
    int resultado = a + b;
    return resultado;
}
```

Um Assembly plausível (levemente simplificado, sem otimizações de compilador) seria:

```asm
soma:
    push rbp                   ; PRÓLOGO
    mov rbp, rsp
    sub rsp, 16

    mov dword [rbp-4], edi     ; a (chega via registrador, Módulo 10) é guardado na stack
    mov dword [rbp-8], esi     ; b (idem) é guardado na stack

    mov eax, dword [rbp-4]     ; PROCESSAMENTO
    add eax, dword [rbp-8]
    mov dword [rbp-12], eax    ; resultado = a + b

    mov eax, dword [rbp-12]    ; SAÍDA: valor de retorno vai em EAX

    mov rsp, rbp               ; EPÍLOGO
    pop rbp
    ret
```

### 9.1 Identificando as três partes

**Entrada (prólogo + recepção de argumentos):**

```asm
push rbp
mov rbp, rsp
sub rsp, 16
mov dword [rbp-4], edi
mov dword [rbp-8], esi
```

O stack frame é montado, e os argumentos `a` e `b`, que chegaram via registradores (`EDI` e `ESI`, adiantando o que o Módulo 10 vai formalizar), são copiados para posições fixas na stack, para poderem ser referenciados como `[rbp-4]` e `[rbp-8]` no restante da função.

**Processamento (o corpo lógico da função):**

```asm
mov eax, dword [rbp-4]
add eax, dword [rbp-8]
mov dword [rbp-12], eax
```

Aqui está a lógica real da função: ler `a`, somar `b`, guardar o resultado em uma terceira posição da stack (`resultado`, em `[rbp-12]`).

**Saída (preparar o retorno + epílogo):**

```asm
mov eax, dword [rbp-12]
mov rsp, rbp
pop rbp
ret
```

O valor a ser retornado é colocado em `EAX` (a convenção padrão para valores de retorno de até 32 bits, como já mencionado no Módulo 3), o stack frame é desfeito, e a execução retorna para quem chamou.

## 10. Processo de leitura para funções com stack frame

Ao encontrar uma função Assembly desconhecida, este processo ajuda a organizá-la mentalmente:

1. **Localizar o prólogo**: `push rbp` / `mov rbp, rsp` / `sub rsp, N` (ou uma variação equivalente). Isso marca o início do frame e revela quanto espaço foi reservado para variáveis locais (`N`).
2. **Localizar o epílogo**: `mov rsp, rbp` / `pop rbp` / `ret` (ou `leave` / `ret`). Isso marca o fim da função.
3. **Tudo entre o prólogo e o epílogo é, em algum sentido, "entrada" (organizar argumentos), "processamento" (a lógica real) ou "saída" (preparar o valor de retorno)**, nessa ordem geral, embora nem sempre estritamente separados.
4. **Identificar os deslocamentos negativos** (`[rbp-4]`, `[rbp-8]`...): são variáveis locais. Quantos deslocamentos distintos aparecem dá uma pista de quantas variáveis locais a função usa.
5. **Identificar os deslocamentos positivos** (`[rbp+16]`...), se houver: são argumentos que vieram pela stack, não por registrador.
6. **Verificar o valor final em `EAX`/`RAX`** antes do epílogo: geralmente é o valor de retorno da função.

## 11. Erros comuns de leitura

- **Confundir "crescer para baixo" com uma direção física absoluta.** "Para baixo" descreve o valor numérico do endereço diminuindo, não uma posição física real na memória RAM. A confusão com a intuição de "empilhar objetos de baixo para cima" vem de misturar duas escalas diferentes (posição visual vs. valor numérico do endereço), como visto na Seção 3.2.
- **Confundir `RSP` com `RBP`.** `RSP` se move constantemente durante a execução da função; `RBP` fica fixo do prólogo ao epílogo. Deslocamentos de variáveis e argumentos são quase sempre relativos a `RBP`, justamente por essa estabilidade.
- **Esquecer que a stack cresce para baixo.** Isso leva a inverter mentalmente o sentido de "acima" e "abaixo": argumentos (empilhados antes do prólogo) ficam em endereços **mais altos** que `RBP` (por isso, deslocamento positivo); variáveis locais (reservadas depois do prólogo) ficam em endereços **mais baixos** (deslocamento negativo).
- **Assumir que todo `[rbp-N]` representa exatamente uma variável de C.** Compiladores podem reorganizar, combinar ou até eliminar variáveis locais durante otimização; o número de deslocamentos distintos é uma pista, não uma garantia exata de correspondência 1 para 1 com o código-fonte.
- **Ignorar o `leave`** por não reconhecer a instrução: lembrar que ela é apenas um atalho para `mov rsp, rbp` seguido de `pop rbp`.

## 12. Tabela-resumo

| Elemento | Papel |
|---|---|
| `RSP` | Aponta sempre para o topo atual da stack; muda constantemente |
| `RBP` | Fixo durante toda a função; serve como referência estável para variáveis e argumentos |
| `push rbp` / `mov rbp, rsp` / `sub rsp, N` | Prólogo: salva o RBP anterior, estabelece o novo frame, reserva espaço |
| `mov rsp, rbp` / `pop rbp` / `ret` (ou `leave` / `ret`) | Epílogo: desfaz o frame, restaura o RBP anterior, retorna |
| `[rbp-N]` | Variável local (deslocamento negativo, dentro do espaço reservado) |
| `[rbp+N]` | Argumento vindo pela stack (deslocamento positivo, acima do frame atual) |
| Stack frame | A região entre `RBP` e `RSP` pertencente à chamada de função atual |

## 13. Exercícios

### Nível 1 — Conceitual

1. Por que `RBP` é uma referência mais estável do que `RSP` para acessar variáveis locais?
2. O que a instrução `leave` substitui, exatamente?
3. Por que argumentos vindos pela stack usam deslocamento positivo, enquanto variáveis locais usam deslocamento negativo?
4. Complete a frase, escolhendo a alternativa correta e justificando: "Cada novo valor empilhado (via `push`) ocupa um endereço ___ que o valor empilhado imediatamente antes dele." (maior / menor)

### Nível 3 — Acompanhar endereços numericamente

Suponha que `RSP = 0x4000` antes de qualquer instrução abaixo:

```asm
push rax     ; RAX = 7
push rbx     ; RBX = 20
pop rcx
push rdx     ; RDX = 99
```

5. Preencha, para cada instrução, o novo valor de `RSP` e o que foi lido ou escrito, seguindo o mesmo formato da tabela da Seção 3.1.
6. Ao final desse trecho, qual endereço `RSP` está apontando, e qual valor está armazenado nesse endereço?

### Nível 5 — Interpretar uma função completa

```asm
dobro:
    push rbp
    mov rbp, rsp
    sub rsp, 8

    mov dword [rbp-4], edi

    mov eax, dword [rbp-4]
    add eax, eax
    mov dword [rbp-8], eax

    mov eax, dword [rbp-8]

    mov rsp, rbp
    pop rbp
    ret
```

7. Identifique o prólogo, o processamento e o epílogo deste trecho.
8. Quantas variáveis locais esta função usa, e em que deslocamentos elas estão?
9. Escreva o código C aproximado que esta função representa.

### Nível 6 — Reconstruir a lógica a partir da stack

```asm
maior:
    push rbp
    mov rbp, rsp
    sub rsp, 8

    mov dword [rbp-4], edi
    mov dword [rbp-8], esi

    mov eax, dword [rbp-4]
    cmp eax, dword [rbp-8]
    jge retorna_a
    mov eax, dword [rbp-8]
    jmp fim
retorna_a:
    mov eax, dword [rbp-4]
fim:

    mov rsp, rbp
    pop rbp
    ret
```

10. Escreva o código C aproximado que esta função representa (dica: revise o Módulo 8, Seção 4, sobre reconhecer `if`/`else`, combinado com o que foi visto aqui sobre argumentos).

---

## 14. Respostas

1. Porque `RSP` muda a cada `push`, `pop`, ou qualquer alocação/desalocação temporária de espaço durante a execução da função, tornando qualquer deslocamento relativo a ele instável. `RBP` é definido uma única vez, no prólogo, e permanece fixo até o epílogo, servindo como um ponto de referência confiável durante toda a função.
2. Substitui as duas instruções `mov rsp, rbp` (descartar o espaço reservado para variáveis locais) seguida de `pop rbp` (restaurar o RBP da função anterior).
3. Porque a stack cresce para endereços mais baixos. Argumentos são empilhados **antes** do prólogo da função ser executado, portanto ficam em endereços mais altos que o `RBP` estabelecido depois (deslocamento positivo). Variáveis locais são reservadas **depois** do prólogo, através de `sub rsp, N`, ocupando endereços mais baixos que `RBP` (deslocamento negativo).
4. **Menor.** Cada `push` decrementa `RSP` antes de escrever o novo valor (Módulo 6, Parte 1), então o endereço do valor recém-empilhado é sempre menor que o do valor anterior, como visto na Seção 3.1.
5. 

| Instrução | Novo `RSP` | O que foi lido/escrito |
|---|---|---|
| `push rax` (RAX=7) | `0x3FF8` | `7` escrito em `0x3FF8` |
| `push rbx` (RBX=20) | `0x3FF0` | `20` escrito em `0x3FF0` |
| `pop rcx` | `0x3FF8` | `RCX` recebe `20`, lido de `0x3FF0` |
| `push rdx` (RDX=99) | `0x3FF0` | `99` escrito em `0x3FF0` (sobrescrevendo o `20` anterior) |

6. Ao final, `RSP = 0x3FF0`, e o valor armazenado nesse endereço é `99` (o `20` que estava lá antes foi sobrescrito pelo último `push`).
7. **Prólogo**: `push rbp` / `mov rbp, rsp` / `sub rsp, 8`, além de `mov dword [rbp-4], edi` (recepção do argumento). **Processamento**: `mov eax, [rbp-4]` / `add eax, eax` / `mov dword [rbp-8], eax`. **Saída/epílogo**: `mov eax, [rbp-8]` / `mov rsp, rbp` / `pop rbp` / `ret`.
8. Duas variáveis locais: uma em `[rbp-4]` (o argumento copiado da stack) e outra em `[rbp-8]` (o resultado do dobro).
9. 
```c
int dobro(int a) {
    int resultado = a + a;
    return resultado;
}
```
10. 
```c
int maior(int a, int b) {
    if (a >= b) {
        return a;
    } else {
        return b;
    }
}
```

---

*Módulo anterior: [Módulo 8 — Controle de Fluxo](./08-controle-de-fluxo.md)*
*Próximo módulo: [Módulo 10 — Funções](./10-funcoes.md)*
