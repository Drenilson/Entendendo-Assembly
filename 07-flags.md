# Módulo 7 — Flags

> **Arquitetura:** x86-64
> **Sistema operacional de referência:** Linux
> **ABI:** System V AMD64
> **Sintaxe:** Intel

## 1. Introdução

No Módulo 3, RFLAGS foi apresentado como um registrador especial, e reencontramos a ideia na Parte 1 do Módulo 6: cada bit individual funciona como um interruptor que anota uma característica do resultado da última operação. No Módulo 6 (Partes 2 e 3), já usamos `cmp` e `test` sem detalhar exatamente o que cada flag significa.

Este módulo formaliza isso. Ao final, você deve conseguir olhar para:

```asm
cmp eax, ebx
je  ...
```

e entender exatamente **por que** essa sequência representa algo parecido com `if (a == b)` em C — e, mais importante, **como generalizar** esse raciocínio para `>`, `<`, `>=`, `<=`, tanto em contextos signed quanto unsigned.

## 2. As quatro flags principais

Vamos focar nas quatro flags mais relevantes para leitura de código gerado por compiladores C:

| Flag | Nome completo | O que indica |
|---|---|---|
| **ZF** | Zero Flag | O resultado da última operação foi zero |
| **SF** | Sign Flag | O resultado da última operação foi negativo (bit mais significativo = 1) |
| **CF** | Carry Flag | Houve "estouro" (carry/borrow) em uma operação **unsigned** |
| **OF** | Overflow Flag | Houve "estouro" em uma operação **signed** |

Existem outras flags em RFLAGS (como AF, PF), mas elas raramente aparecem no raciocínio de código C compilado comum — vamos deixá-las de fora por ora, focando em objetivo pedagógico: ler e interpretar.

## 3. ZF — Zero Flag

### O que é

`ZF = 1` quando o resultado da última operação que afeta flags foi exatamente zero. `ZF = 0` em qualquer outro caso.

### Exemplo

```asm
mov eax, 5
sub eax, 5     ; EAX = 0 → ZF = 1

mov eax, 5
sub eax, 3     ; EAX = 2 → ZF = 0
```

### Relação com `cmp`

Como vimos no Módulo 6, `cmp a, b` calcula `a - b` e descarta o resultado, mas atualiza ZF. Logo:

```asm
cmp eax, ebx    ; calcula EAX - EBX (descartado)
                 ; se EAX == EBX, o resultado é 0 → ZF = 1
                 ; se EAX != EBX, o resultado é diferente de 0 → ZF = 0
```

> **ZF = 1 depois de `cmp a, b` significa "a é igual a b".** Essa é a base de `je` (*jump if equal*), que veremos no Módulo 8.

## 4. SF — Sign Flag

### O que é

`SF` copia o bit mais significativo do resultado da última operação — ou seja, reflete se o resultado, interpretado como signed (Módulo 4, Seção 5), é negativo.

### Exemplo

```asm
mov eax, 3
sub eax, 10     ; EAX = -7 → SF = 1 (resultado negativo)

mov eax, 10
sub eax, 3      ; EAX = 7 → SF = 0 (resultado positivo)
```

### Relação com `cmp`

```asm
cmp eax, ebx    ; calcula EAX - EBX
                 ; se EAX < EBX (signed), o resultado é negativo → SF = 1
                 ; se EAX >= EBX (signed), o resultado é >= 0 → SF = 0
```

> **Cuidado:** SF sozinha só confiavelmente indica "menor que", em comparações signed, **quando não há overflow**. Vamos ver por quê na Seção 6.

## 5. CF — Carry Flag

### O que é

`CF = 1` quando uma operação aritmética gera um "vai-um" (carry) que não coube no tamanho do registrador de destino, tratando os valores como **unsigned**. Em subtrações, CF funciona como um "empréstimo" (*borrow*): `CF = 1` quando o primeiro operando, tratado como unsigned, é **menor** que o segundo.

### Exemplo — soma que estoura (unsigned)

```asm
mov al, 0xFF        ; AL = 255 (unsigned)
add al, 1             ; 255 + 1 = 256, mas AL só tem 8 bits → AL = 0x00, CF = 1
```

O resultado matemático real (256) não coube em 8 bits — o bit "extra" é registrado em CF, mesmo que o valor em AL pareça, isoladamente, apenas "zero".

### Exemplo — `cmp` e CF em comparações unsigned

```asm
mov al, 3
cmp al, 10      ; 3 - 10, tratando como unsigned: 3 é MENOR que 10 → CF = 1
```

> **CF = 1 depois de `cmp a, b` significa "a é menor que b", em uma leitura unsigned.** Essa é a base de instruções como `jb` (*jump if below*), usada para comparações **unsigned** — diferente de `jl` (*jump if less*), usada para comparações **signed**, que veremos no Módulo 8.

## 6. OF — Overflow Flag

### O que é

`OF = 1` quando o resultado de uma operação aritmética **signed** não coube corretamente no tamanho do destino — ou seja, quando o resultado matemático real ultrapassa a faixa representável em complemento de dois para aquele tamanho (Módulo 4, Seção 5.1).

### Exemplo — overflow signed

```asm
mov al, 127      ; AL = 127 (o maior valor signed possível em 8 bits)
add al, 1          ; resultado matemático: 128, mas 128 não é representável em 8 bits signed
                    ; AL = 1000 0000 = -128 (em complemento de dois) → OF = 1
```

Aqui, `AL` passou de `127` (o máximo signed) para `-128` — um resultado absurdo se interpretado como "somar 1 a um número positivo". A flag `OF = 1` sinaliza exatamente essa inconsistência: o resultado, olhando os bits, é "válido" (é um padrão de bits legítimo), mas **não representa corretamente** a soma matemática real ao ser lido como signed.

### Por que CF e OF são diferentes

Essa é uma das distinções mais importantes deste módulo, então vale isolar:

> **CF detecta problemas em interpretações unsigned. OF detecta problemas em interpretações signed.** A mesma operação pode ter CF = 1 e OF = 0, ou CF = 0 e OF = 1, ou qualquer outra combinação — são checagens **independentes**, porque "estourar como unsigned" e "estourar como signed" são condições diferentes.

Veja o exemplo da Seção 5 revisitado, olhando as duas flags:

```asm
mov al, 0xFF        ; AL = 255 (unsigned) ou -1 (signed)
add al, 1             ; resultado: AL = 0x00
```

- Como **unsigned**: `255 + 1 = 256`, não cabe em 8 bits → **CF = 1** (estourou).
- Como **signed**: `-1 + 1 = 0`, que cabe perfeitamente em 8 bits signed → **OF = 0** (não estourou).

A mesma instrução, a mesma sequência de bits resultante (`0x00`), mas **interpretações diferentes levam a conclusões diferentes sobre "se deu overflow"**. Isso reforça, mais uma vez, a ideia central do Módulo 4: os bits não sabem se são signed ou unsigned — as flags refletem **ambas as possibilidades simultaneamente**, e cabe à instrução seguinte (um `jb` ou um `jl`, por exemplo) escolher qual interpretação importa.

## 7. Juntando tudo: `cmp` seguido de decisão

Agora que conhecemos as quatro flags, podemos formalizar como `cmp a, b` permite decidir entre `==`, `!=`, `>`, `<`, `>=`, `<=` — tanto em contextos **signed** quanto **unsigned**.

### 7.1 Igualdade — independe de signed/unsigned

```
a == b   →   ZF = 1
a != b   →   ZF = 0
```

Igualdade é a única comparação que **não depende** de interpretar os valores como signed ou unsigned — se os bits são idênticos, `a - b` é zero em qualquer interpretação.

### 7.2 Comparações unsigned — usam CF

```
a < b  (unsigned)   →   CF = 1
a >= b (unsigned)   →   CF = 0
a > b  (unsigned)   →   CF = 0 e ZF = 0
a <= b (unsigned)   →   CF = 1 ou ZF = 1
```

### 7.3 Comparações signed — usam SF e OF juntas

Aqui está o ponto mais sutil do módulo. Não basta olhar SF isoladamente — é preciso combinar SF com OF, porque overflow "engana" o sinal do resultado.

```
a < b  (signed)   →   SF != OF   (SF e OF são diferentes entre si)
a >= b (signed)   →   SF == OF
a > b  (signed)   →   SF == OF e ZF = 0
a <= b (signed)   →   SF != OF ou ZF = 1
```

### 7.4 Por que SF sozinha não basta (o caso de overflow)

Vamos construir um exemplo que demonstra exatamente por que a regra da Seção 7.3 exige combinar SF e OF, em vez de usar SF isoladamente.

```asm
mov al, -128       ; AL = 1000 0000 (o menor valor signed possível em 8 bits)
mov bl, 1
cmp al, bl          ; calcula AL - BL = -128 - 1 = -129 (matematicamente)
```

O resultado matemático real, `-129`, **não cabe** em 8 bits signed (a faixa é -128 a 127). Então o que a CPU realmente calcula, em binário puro, "estoura":

```
  1000 0000   (-128)
-    0000 0001  (1)
------------
  0111 1111    (resultado em bits: 127!)
```

O resultado em bits é `0111 1111`, que como signed é **127** — um número **positivo**! Então `SF = 0` (bit de sinal é 0). Se olhássemos só para SF, concluiríamos erroneamente que `AL >= BL` (já que o "resultado" pareceu não-negativo). Mas sabemos que `-128 < 1` deveria ser verdadeiro.

É exatamente aqui que `OF` entra: como o resultado real matematicamente deveria ser negativo (`-129`), mas o bit de sinal do resultado calculado deu positivo (`0`), a CPU sinaliza **OF = 1** — "o sinal deste resultado não é confiável, houve overflow signed". A regra `SF != OF` captura exatamente esse caso: `SF = 0` e `OF = 1` são diferentes entre si, então a regra conclui corretamente "a < b", mesmo com SF "mentindo" sozinha.

> Você não precisa decorar a mecânica bit a bit desse exemplo — o que precisa fixar é a **conclusão**: comparações signed corretas sempre combinam SF e OF, nunca confiam em SF isoladamente. Compiladores (e, mais adiante no curso, as próprias instruções de jump condicional) já embutem essa combinação automaticamente — mas, ao ler Assembly gerado à mão ou em nível muito baixo, reconhecer o padrão `SF != OF` / `SF == OF` é o que permite confirmar que uma comparação signed está sendo feita corretamente.

## 8. Tabela-resumo

| Comparação desejada | Signed | Unsigned |
|---|---|---|
| `a == b` | ZF = 1 | ZF = 1 |
| `a != b` | ZF = 0 | ZF = 0 |
| `a < b` | SF != OF | CF = 1 |
| `a >= b` | SF == OF | CF = 0 |
| `a > b` | SF == OF e ZF = 0 | CF = 0 e ZF = 0 |
| `a <= b` | SF != OF ou ZF = 1 | CF = 1 ou ZF = 1 |

Essa tabela é exatamente o que, no Módulo 8, vai se traduzir nos pares de instruções `jl`/`jb`, `jge`/`jae`, `jg`/`ja`, `jle`/`jbe` — cada uma "hard-codeando" uma dessas combinações de flags.

## 9. Exemplo prático integrado

```asm
mov eax, 5
mov ebx, 8
cmp eax, ebx     ; calcula 5 - 8 = -3
```

Vamos analisar as flags resultantes, assumindo uma comparação de inteiros de 32 bits comuns (sem overflow, já que os valores são pequenos):

- **ZF**: o resultado (-3) não é zero → `ZF = 0`.
- **SF**: o resultado (-3) é negativo → `SF = 1`.
- **OF**: não houve overflow signed (a subtração de dois números pequenos e positivos nunca estoura a faixa de 32 bits) → `OF = 0`.
- **CF**: tratando `5` e `8` como unsigned, `5 < 8`, então há "empréstimo" → `CF = 1`.

Agora aplicamos a Seção 8:

- **Signed**: `SF != OF` → `1 != 0` → verdadeiro → `EAX < EBX` (signed). Correto: 5 < 8.
- **Unsigned**: `CF = 1` → `EAX < EBX` (unsigned). Também correto: 5 < 8 (como unsigned também).

Neste exemplo específico, as duas interpretações concordam — o que é o caso mais comum quando não há valores negativos nem overflow envolvidos. As duas interpretações **divergem** justamente nos casos com números negativos ou próximos dos limites de faixa, como vimos na Seção 7.4.

## 10. Exercícios

### Nível 1 — Conceitual

1. O que significa `ZF = 1` após uma instrução `cmp`?
2. Qual é a diferença entre o que `CF` detecta e o que `OF` detecta?
3. Por que não é seguro usar apenas `SF` para decidir "menor que" em comparações signed?

### Nível 2 — Interpretar flags a partir de instruções

Para cada `cmp`, determine os valores de ZF e SF (assuma que não há overflow signed, a menos que indicado):

4. `mov eax, 10` / `mov ebx, 10` / `cmp eax, ebx`
5. `mov eax, 4` / `mov ebx, 9` / `cmp eax, ebx`
6. `mov eax, 9` / `mov ebx, 4` / `cmp eax, ebx`

### Nível 3 — Aplicar a tabela de comparação

Para o trecho abaixo, determine se a condição `EAX > EBX` (signed) é verdadeira ou falsa, usando a regra da Seção 8 (não apenas comparando os números diretamente — pratique usar SF e OF):

```asm
mov eax, 20
mov ebx, 5
cmp eax, ebx
```

7. Quais são os valores de SF, OF e ZF após este `cmp`?
8. Segundo a regra `SF == OF e ZF = 0`, a condição `EAX > EBX` é verdadeira?

---

## 11. Respostas

1. Significa que o resultado da subtração implícita (`operando1 - operando2`) foi exatamente zero — ou seja, os dois operandos comparados eram iguais.
2. `CF` detecta estouro (carry/borrow) considerando os operandos como **unsigned** — é relevante para comparações e aritmética sem sinal. `OF` detecta estouro considerando os operandos como **signed** — é relevante para aritmética e comparações com sinal, indicando quando o resultado ultrapassou a faixa representável em complemento de dois.
3. Porque, em caso de overflow signed, o bit de sinal do resultado pode "mentir" — um resultado que deveria ser negativo (matematicamente) pode aparecer com o bit de sinal em 0 (parecendo positivo), e vice-versa. Combinar SF com OF (checando se são iguais ou diferentes) corrige esse problema, detectando corretamente a relação de ordem mesmo quando houve overflow.
4. `10 - 10 = 0` → `ZF = 1`, `SF = 0` (resultado não é negativo).
5. `4 - 9 = -5` → `ZF = 0`, `SF = 1` (resultado é negativo).
6. `9 - 4 = 5` → `ZF = 0`, `SF = 0` (resultado é positivo).
7. `20 - 5 = 15`. `ZF = 0` (resultado não é zero). `SF = 0` (resultado positivo). Como os dois valores são pequenos e positivos, não há overflow signed possível aqui → `OF = 0`.
8. `SF == OF` → `0 == 0` → verdadeiro. E `ZF = 0`. Logo, a condição completa (`SF == OF` e `ZF = 0`) é satisfeita → **verdadeiro**, `EAX > EBX`. Isso confere com a comparação direta dos números (20 > 5), como esperado em um caso sem overflow.

---

*Módulo anterior: [Módulo 6 — Instruções Fundamentais (Parte 3)](./06-instrucoes-fundamentais-parte3.md)*
*Próximo módulo: [Módulo 8 — Controle de Fluxo](./08-controle-de-fluxo.md)*
