# Módulo 3 — Registradores

> **Arquitetura:** x86-64
> **Sistema operacional de referência:** Linux
> **ABI:** System V AMD64
> **Sintaxe:** Intel

## 1. Introdução

No Módulo 2, os registradores foram apresentados como a "mesa de trabalho" da CPU: espaços de armazenamento internos, extremamente rápidos, onde os valores ficam enquanto são manipulados.

Este módulo não pretende ser uma tabela para decorar. O objetivo é entender **por que** existem vários registradores, **por que** eles têm tamanhos e subdivisões diferentes, e **como** eles aparecem, na prática, em código Assembly.

## 2. Quantos registradores existem?

Em x86-64, os principais registradores de uso geral são:

```
RAX  RBX  RCX  RDX  RSI  RDI  RBP  RSP  R8  R9  R10  R11  R12  R13  R14  R15
```

São 16 registradores de 64 bits. Além deles, existem dois registradores especiais que já mencionamos e vamos revisitar aqui: `RIP` (instruction pointer) e `RFLAGS` (flags).

Note o prefixo `R` — ele indica "64 bits" (de *Register*, na extensão x86-64). Os registradores mais antigos, de 32 bits, tinham prefixo `E` (`EAX`, `EBX`...), herdados da era 32-bit (x86). Isso não é coincidência — é o ponto central da próxima seção.

## 3. Por que um registrador tem "partes"?

### 3.1 O caso RAX / EAX / AX / AL

Considere `RAX`. Ele tem 64 bits. Mas você pode acessar apenas uma **parte** dele:

```
RAX (64 bits)
 └── EAX (32 bits menos significativos)
      └── AX (16 bits menos significativos)
           ├── AH (byte alto de AX, bits 8-15)
           └── AL (byte baixo de AX, bits 0-7)
```

Visualmente, pensando em RAX como 8 bytes lado a lado:

```
  byte7   byte6    byte5   byte4   byte3    byte2   byte1   byte0
┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┐
│       │       │       │       │       │       │   AH   │  AL   │
└──────┴──────┴──────┴──────┴──────┴──────┴──────┴──────┘
└─────────────────────┬──────────────────┘└──────┬──────┘
                         EAX                        AX
└────── RAX ─────────────────────────────────────────────┘
```

**Por que isso existe?** Motivo histórico e prático: a arquitetura x86 evoluiu ao longo de décadas.

- Nos anos 1970-80, os processadores 8086 trabalhavam com registradores de **16 bits** (`AX`, `BX`, `CX`, `DX`), e cada um podia ser dividido em dois bytes (`AH`/`AL`, `BH`/`BL`...).
- Nos anos 1980-90, surgiu a extensão para **32 bits** (`EAX`, `EBX`...) — o `E` vem de *Extended*. Os registradores de 32 bits **contêm** os antigos registradores de 16 bits em sua metade inferior, mantendo compatibilidade com código antigo.
- Nos anos 2000, surgiu a extensão para **64 bits** (`RAX`, `RBX`...) — o `R` vem de *Register*. Novamente, os registradores de 64 bits **contêm** os de 32 bits em sua metade inferior.

O resultado é que hoje, ao escrever `AL`, `AX`, `EAX` ou `RAX`, você está acessando **partes do mesmo espaço físico**, apenas com tamanhos diferentes. Isso permite que um programa trabalhe com dados pequenos (um único byte, `AL`) sem precisar mover todo um registrador de 64 bits.

### 3.2 Consequência prática importante

Se você escrever:

```asm
mov rax, 0x1122334455667788
mov al,  0xFF
```

Depois da segunda instrução, apenas o **byte menos significativo** de RAX foi alterado. RAX passa a valer `0x11223344556677FF` — o resto permanece como estava.

> Isso é algo que confunde muita gente ao começar. Fixe: **modificar uma "parte" de um registrador não apaga o resto — apenas altera os bits correspondentes àquela parte.**

Existe uma exceção importante: em modo 64 bits, escrever em um registrador de **32 bits** (como `EAX`) **zera automaticamente** a metade superior de 64 bits. Ou seja:

```asm
mov rax, 0x1122334455667788
mov eax, 0xFF
```

Depois disso, RAX vale `0x00000000000000FF` — a parte superior foi zerada. Isso **não** acontece ao escrever em `AX` ou `AL` (essas operações preservam o restante do registrador). Essa é uma particularidade específica de x86-64 que vale a pena guardar, pois aparece com frequência em código gerado por compiladores.

### 3.3 O mesmo padrão se aplica a outros registradores

`RBX`/`EBX`/`BX`/`BL` (e `BH`), `RCX`/`ECX`/`CX`/`CL` (e `CH`), `RDX`/`EDX`/`DX`/`DL` (e `DH`) seguem exatamente o mesmo padrão de RAX. Já `RSI`, `RDI`, `RBP`, `RSP` e `R8`-`R15` também têm versões de 32, 16 e 8 bits, mas com nomenclatura um pouco diferente (ver tabela na Seção 5).

## 4. Registradores de uso geral: o que cada um "costuma" representar

Todos os registradores de uso geral podem, tecnicamente, guardar qualquer valor — não há uma regra rígida imposta pelo hardware. Mas **convenções de uso** (adotadas por compiladores e pela ABI) fazem certos registradores aparecerem com papéis típicos. Conhecer essas convenções ajuda muito na leitura:

| Registrador | Papel comum (convenção, não regra fixa) |
|---|---|
| `RAX` | Acumulador; costuma guardar o **valor de retorno** de uma função |
| `RBX` | Uso geral; historicamente "base"; frequentemente preservado entre chamadas |
| `RCX` | Uso geral; historicamente usado como **contador** (ex.: em laços) |
| `RDX` | Uso geral; frequentemente usado em operações de multiplicação/divisão junto com RAX |
| `RSI` | *Source Index* — frequentemente "origem" em operações de cópia; hoje também usado para argumentos de função |
| `RDI` | *Destination Index* — frequentemente "destino" em operações de cópia; hoje também usado para argumentos de função |
| `RBP` | *Base Pointer* — costuma marcar a "base" do frame da stack de uma função |
| `RSP` | *Stack Pointer* — sempre aponta para o topo atual da stack |
| `R8`–`R15` | Registradores adicionais introduzidos em x86-64, sem papel histórico fixo; muito usados para argumentos e valores temporários |

> Vamos ver, no Módulo 10 (Funções), que `RDI`, `RSI`, `RDX`, `RCX`, `R8`, `R9` têm um papel bem específico na **convenção de chamada (calling convention)** do Linux/System V: são os primeiros seis registradores usados para passar argumentos para uma função. Por ora, apenas registre esse nome — vamos aprofundar depois.

## 5. Tabela de subdivisões (referência, não para decorar)

| 64 bits | 32 bits | 16 bits | 8 bits (baixo) | 8 bits (alto) |
|---|---|---|---|---|
| RAX | EAX | AX | AL | AH |
| RBX | EBX | BX | BL | BH |
| RCX | ECX | CX | CL | CH |
| RDX | EDX | DX | DL | DH |
| RSI | ESI | SI | SIL | — |
| RDI | EDI | DI | DIL | — |
| RBP | EBP | BP | BPL | — |
| RSP | ESP | SP | SPL | — |
| R8 | R8D | R8W | R8B | — |
| R9 | R9D | R9W | R9B | — |
| ... | ... | ... | ... | — |
| R15 | R15D | R15W | R15B | — |

> Note que `RSI`, `RDI`, `RBP`, `RSP` **não** têm um byte "alto" acessível (não existe `SIH`, por exemplo) — apenas `AX`, `BX`, `CX`, `DX` têm essa divisão histórica em alto/baixo, herdada dos registradores de 16 bits originais do 8086.

O importante aqui não é memorizar a tabela inteira agora — é **saber que ela existe e por que**, e voltar a ela quando necessário durante a leitura de código real.

## 6. RIP — revisão

Já vimos RIP no Módulo 2: é o *Instruction Pointer*, que sempre aponta para o endereço da próxima instrução a ser executada. Diferente dos registradores de uso geral, você **não** costuma manipular RIP diretamente com `mov` — ele é alterado implicitamente por instruções de fluxo (`jmp`, `call`, `ret`, saltos condicionais), que veremos no Módulo 6 e no Módulo 8.

## 7. RFLAGS — introdução

`RFLAGS` é um registrador especial de 64 bits onde **cada bit individual** representa uma condição sobre o resultado da última operação executada. Por exemplo: "o resultado foi zero?", "houve overflow?", "o resultado foi negativo?".

Neste módulo, apenas registramos sua existência — ele será o assunto central do **Módulo 7 (Flags)**, onde veremos como instruções como `cmp` o modificam, e como instruções de desvio condicional (`je`, `jg`...) o leem para decidir o fluxo de execução.

## 8. Exemplo prático integrado

Considere o seguinte trecho:

```asm
mov rax, 5      ; RAX = 5
mov bl, 10      ; BL (parte baixa de RBX) = 10, resto de RBX inalterado
add al, bl      ; AL = AL + BL → AL = 15, resto de RAX inalterado
```

Vamos aplicar o raciocínio:

1. `mov rax, 5` — o registrador completo de 64 bits `RAX` recebe o valor 5. Como é uma instrução em 64 bits explícita, o valor 5 ocupa apenas os bits baixos; o resto é zero.
2. `mov bl, 10` — apenas o byte baixo de `RBX` (`BL`) recebe o valor 10. O restante de `RBX` não é alterado por esta instrução (pode conter lixo de uso anterior, ou zero, dependendo do contexto).
3. `add al, bl` — soma o byte baixo de RAX (`AL`, que vale 5) com o byte baixo de RBX (`BL`, que vale 10), e grava o resultado (15) em `AL`. Apenas esse byte de RAX é afetado — os demais bytes de RAX permanecem como estavam.

Esse tipo de raciocínio — "qual parte do registrador está sendo lida ou escrita, e o que acontece com o resto" — é exatamente o tipo de leitura cuidadosa que este curso quer desenvolver.

## 9. Exercícios

### Nível 1 — Conceitual

1. Por que `EAX` é considerado "parte de" `RAX`, e não um registrador totalmente independente?
2. O que diferencia, historicamente, o surgimento de `EAX` e de `RAX`?
3. Verdadeiro ou falso: escrever em `AL` apaga o restante de `RAX`. Justifique.
4. Verdadeiro ou falso: escrever em `EAX` apaga a metade superior de `RAX`. Justifique.
5. Quais registradores possuem uma divisão em byte "alto" (`AH`, `BH`, `CH`, `DH`) e quais não possuem?

### Nível 2 — Interpretar instruções

Para o trecho abaixo, explique o efeito de cada instrução sobre os registradores envolvidos:

```asm
mov rcx, 0xAABBCCDDEEFF0011
mov cx, 0x9999
mov ch, 0x00
```

---

## 10. Respostas

1. Porque fisicamente `EAX` ocupa os 32 bits menos significativos do mesmo espaço de armazenamento de `RAX`. Não são dois registradores separados — é o mesmo registrador acessado com uma "janela" menor.
2. `EAX` (e os demais registradores com prefixo `E`) surgiu com a extensão para 32 bits do x86 (anos 1980-90). `RAX` (prefixo `R`) surgiu bem depois, com a extensão para 64 bits (x86-64, anos 2000). `RAX` contém `EAX` em sua metade inferior, assim como `EAX` continha `AX` na sua.
3. **Falso.** Escrever em `AL` altera apenas o byte menos significativo de `RAX`; os demais 7 bytes permanecem inalterados.
4. **Verdadeiro.** Essa é uma particularidade específica do modo 64 bits: escrever em um registrador de 32 bits (como `EAX`) zera automaticamente a metade superior de 64 bits do registrador correspondente (`RAX`). Isso não acontece ao escrever em versões de 16 ou 8 bits.
5. Apenas `AX`, `BX`, `CX` e `DX` (dando origem a `AH`/`AL`, `BH`/`BL`, `CH`/`CL`, `DH`/`DL`) possuem essa divisão em byte alto e baixo — é uma característica herdada dos registradores de 16 bits originais do 8086. `SI`, `DI`, `BP`, `SP` e `R8`-`R15` não possuem um byte "alto" endereçável separadamente; apenas o byte baixo (`SIL`, `DIL`, `BPL`, `SPL`, `R8B`...) é acessível diretamente.
6. Para o trecho do Nível 2:
   - `mov rcx, 0xAABBCCDDEEFF0011` — RCX (64 bits) recebe o valor completo `0xAABBCCDDEEFF0011`.
   - `mov cx, 0x9999` — apenas os 16 bits menos significativos de RCX (`CX`) são alterados. RCX passa a valer `0xAABBCCDDEEFF9999` (os bytes superiores permanecem inalterados, pois escrever em uma parte de 16 ou 8 bits **não** zera o restante — diferente do que ocorre com escrita em 32 bits).
   - `mov ch, 0x00` — apenas o byte alto de `CX` (`CH`, que corresponde ao segundo byte menos significativo de RCX) é zerado. RCX passa a valer `0xAABBCCDDEEFF0099` (o byte baixo `CL`, que valia `0x99`, permanece; o byte `CH`, que valia `0x99`, torna-se `0x00`).

---

*Módulo anterior: [Módulo 2 — Modelo Mental da CPU](./02-modelo-mental-da-cpu.md)*

*Próximo módulo: [Módulo 4 — Dados](./04-dados.md)*
