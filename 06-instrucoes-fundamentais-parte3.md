# Módulo 6 — Instruções Fundamentais (Parte 3)

> **Arquitetura:** x86-64
> **Sistema operacional de referência:** Linux
> **ABI:** System V AMD64
> **Sintaxe:** Intel

## 1. Introdução

Fechando o Módulo 6, veremos dois grupos finais de instruções: os **deslocamentos de bits** (`shl`, `shr`, `sar`) e as **instruções de comparação** (`cmp`, `test`).

Este último grupo é especialmente importante: `cmp` e `test` são o ponto de partida direto para o **Módulo 7 (Flags)** — sem entender o que elas fazem, não é possível entender como `if`, `while` e `for` funcionam em Assembly. Trate esta parte como uma ponte deliberada para o próximo módulo, e — seguindo o padrão que vínhamos usando na Parte 2 — vamos manter vários exemplos por instrução e comparações lado a lado, em vez de uma única passada rápida por cada uma.

## 2. Deslocamentos de bits: a ideia geral

Deslocar bits significa "empurrar" todos os bits de um valor para a esquerda ou para a direita, preenchendo os espaços vazios de alguma forma (que varia conforme a instrução). Vale visualizar isso como uma fileira de bits que desliza:

```
Antes:    0000 1011
Shift ←:  0001 0110   (deslocado 1 posição para a esquerda)
```

Cada instrução deste grupo se diferencia por **como** o espaço vazio é preenchido, e é isso que vamos detalhar — inclusive comparando as três com o mesmo valor de entrada, para deixar a diferença impossível de esquecer.

## 3. `shl` — deslocamento lógico à esquerda

### O que faz

Desloca todos os bits do operando para a **esquerda**, preenchendo com **zeros** pela direita. O bit que "sai" pela esquerda vai para a flag CF (Carry Flag) — e é, de fato, perdido do valor em si, restando apenas registrado ali.

### Sintaxe geral

```asm
shl destino, quantidade
```

### Exemplo comentado 1 — uso básico

```asm
mov al, 0000 0011b   ; AL = 3 (usando notação binária com sufixo 'b', comum em NASM)
shl al, 1              ; desloca 1 posição para a esquerda
                        ; AL = 0000 0110 = 6
```

```
Antes:   0000 0011  (3)
Depois:  0000 0110  (6)
```

### Relação com multiplicação

Deslocar 1 bit para a esquerda **equivale a multiplicar por 2**. Deslocar `n` bits equivale a multiplicar por `2^n`. Compiladores frequentemente usam `shl` no lugar de `imul` quando o multiplicador é uma potência de 2, por ser mais rápido:

```asm
mov eax, 5
shl eax, 3      ; EAX = 5 * (2^3) = 5 * 8 = 40
```

### Exemplo comentado 2 — quando um bit "sai" pela esquerda (overflow do deslocamento)

Veja o que acontece quando o valor não "cabe" mais depois do deslocamento — usando um registrador de 8 bits para deixar isso visível com poucos dígitos:

```asm
mov al, 1100 0000b   ; AL = 192 (unsigned)
shl al, 2              ; desloca 2 posições para a esquerda
```

Fazendo o deslocamento bit a bit:

```
Antes:        1100 0000
Shift 1×:    [1]1000 0000    ← o bit que "sairia" primeiro é anotado (seria CF nesse passo)
Shift 2×:    [1]0000 0000    ← o bit seguinte também sai
```

Como `AL` só tem 8 bits, os dois bits mais significativos originais (`1` e `1`) são simplesmente **descartados** — não existe "onde" eles iriam. O resultado final é `AL = 0000 0000 = 0`, e CF reflete o **último** bit que saiu (não um histórico de todos eles). Isso ilustra por que `shl` não é uma multiplicação "seguramente reversível" — se bits significativos saem, essa informação se perde de verdade, e o valor resultante pode não ter relação óbvia com "o número original vezes alguma potência de 2".

### Erro comum de interpretação

Esquecer que bits que "saem" pela esquerda são **perdidos** (exceto pelo registro em CF, que só guarda o último bit deslocado para fora, não todos) — se o deslocamento fizer o valor "estourar" o tamanho do registrador, esses bits mais significativos simplesmente desaparecem do valor em si, como no exemplo acima.

## 4. `shr` — deslocamento lógico à direita

### O que faz

Desloca todos os bits do operando para a **direita**, preenchendo com **zeros** pela esquerda. O bit que "sai" pela direita vai para CF.

### Sintaxe geral

```asm
shr destino, quantidade
```

### Exemplo comentado

```asm
mov al, 0000 1100b   ; AL = 12
shr al, 2              ; desloca 2 posições para a direita
                        ; AL = 0000 0011 = 3
```

```
Antes:   0000 1100  (12)
Depois:  0000 0011  (3)
```

### Relação com divisão

Deslocar 1 bit para a direita **equivale a dividir por 2** — mas apenas para valores **unsigned** (sem sinal). Isso é importante: `shr` sempre preenche com zeros à esquerda, então se o valor original for interpretado como signed e for negativo, o resultado **não** corresponde a uma divisão signed correta.

```asm
mov eax, 20
shr eax, 2       ; EAX = 20 / (2^2) = 20 / 4 = 5  (correto, valor positivo)
```

### Erro comum de interpretação

Usar `shr` mentalmente como "divisão" para qualquer valor, inclusive negativos. Para deslocamento que preserva o sinal corretamente em valores negativos, existe a próxima instrução: `sar`.

## 5. `sar` — deslocamento aritmético à direita

### O que faz

Assim como `shr`, desloca bits para a direita — mas, em vez de preencher com zeros, **preenche repetindo o bit de sinal** (o bit mais significativo original). Isso preserva corretamente o sinal de valores negativos em complemento de dois.

### Sintaxe geral

```asm
sar destino, quantidade
```

### Exemplo comentado 1 — valor positivo (mesmo resultado que `shr`)

```asm
mov al, 0000 1100b   ; AL = 12
sar al, 2              ; AL = 0000 0011 = 3   (igual a shr, pois o bit de sinal era 0)
```

### Exemplo comentado 2 — valor negativo (aqui a diferença aparece)

```asm
mov al, 1111 1000b   ; AL = -8 (signed, complemento de dois, 8 bits)
sar al, 1              ; preenche com o bit de sinal (1)
                        ; AL = 1111 1100 = -4
```

```
Antes:   1111 1000  (-8, signed)
Depois:  1111 1100  (-4, signed)   ← preenchido com 1s, preservando o sinal
```

### Exemplo comentado 3 — as três instruções lado a lado, mesmo valor de entrada

Este é o exemplo mais importante da seção — vale mais do que qualquer explicação em texto. Partindo do mesmo byte `1111 1000` (-8 signed, ou 248 unsigned) e deslocando 1 posição em cada instrução:

```
Valor original:        1111 1000   (-8 signed / 248 unsigned)

shl al, 1  →           1111 0000   (bit que sai: 1 → CF=1; preenche com 0 pela direita)
shr al, 1  →           0111 1100   (bit que sai: 0 → CF=0; preenche com 0 pela esquerda)
sar al, 1  →           1111 1100   (bit que sai: 0 → CF=0; preenche com o bit de sinal, 1, pela esquerda)
```

Repare como `shr` e `sar` partem do **mesmo bit que sai** (o bit menos significativo, `0`, então CF é igual nos dois casos aqui), mas preenchem o lado oposto de formas **diferentes** — é exatamente aí que mora a diferença entre as duas: `shr` sempre traz `0` pela esquerda; `sar` traz uma cópia do bit de sinal original (`1`, nesse exemplo, porque o valor era negativo). Se o valor original fosse positivo (bit de sinal `0`), `shr` e `sar` produziriam **exatamente o mesmo resultado** — a diferença só aparece quando o bit de sinal é `1`.

### A regra prática

| Instrução | Preenchimento | Uso típico |
|---|---|---|
| `shl` | zeros (pela direita) | multiplicação por potência de 2 (signed ou unsigned) |
| `shr` | zeros (pela esquerda) | divisão por potência de 2, **apenas unsigned** |
| `sar` | bit de sinal (pela esquerda) | divisão por potência de 2, **signed** |

### Erro comum de interpretação

Achar que `shr` e `sar` são intercambiáveis. Compiladores escolhem entre uma e outra **dependendo do tipo declarado em C** (`unsigned int` gera `shr`; `int` gera `sar`) — reconhecer qual das duas está sendo usada é, na prática, uma pista sobre a assinatura (signed/unsigned) do valor original em C. Um segundo erro, específico de `sar`: assumir que ela "sempre produz o mesmo resultado que `shr`" só porque isso é verdade para valores positivos — como visto no Exemplo 3 acima, a diferença só aparece com o bit de sinal ligado, então testar `sar` apenas com números positivos pode esconder esse comportamento por engano.

## 6. `cmp` — comparação

Chegamos à instrução mais importante desta parte, e uma das mais importantes do curso inteiro.

### O que faz

`cmp` **subtrai** o segundo operando do primeiro — exatamente como `sub` faria — mas **descarta o resultado da subtração**. O único efeito real de `cmp` é **atualizar as flags** de RFLAGS com base no que teria acontecido nessa subtração.

### Sintaxe geral

```asm
cmp operando1, operando2
```

Conceitualmente: calcula `operando1 - operando2`, mas não guarda o resultado em lugar nenhum — só atualiza ZF, SF, CF, OF (entre outras).

### Por que isso é útil?

Porque instruções de desvio condicional (`je`, `jg`, `jl`... — Módulo 7 e Módulo 8) **leem essas flags** para decidir se devem ou não desviar o fluxo de execução. `cmp` é, na prática, o "preparo" que antecede quase todo `if`, `while` ou `for` em Assembly.

### Exemplo comentado 1 — operandos iguais

```asm
mov eax, 5
mov ebx, 5
cmp eax, ebx     ; calcula 5 - 5 = 0 (descartado); ZF é setada para 1, pois o resultado foi zero
```

### Exemplo comentado 2 — primeiro operando maior

```asm
mov eax, 10
mov ebx, 3
cmp eax, ebx     ; calcula 10 - 3 = 7 (descartado); ZF = 0; SF = 0 (resultado positivo)
```

### Exemplo comentado 3 — primeiro operando menor

```asm
mov eax, 3
mov ebx, 10
cmp eax, ebx     ; calcula 3 - 10 = -7 (descartado); ZF = 0; SF = 1 (resultado negativo)
```

### Tabela de cenários — o que cada comparação produz nas flags principais

Vamos formalizar exatamente o significado de cada flag no Módulo 7, mas já vale ter esta referência rápida com os quatro cenários mais comuns, todos usando `cmp a, b`:

| Cenário | Resultado de `a - b` | ZF | SF |
|---|---|---|---|
| `a == b` | 0 | 1 | 0 |
| `a > b` (ambos positivos, sem overflow) | positivo | 0 | 0 |
| `a < b` (ambos positivos, sem overflow) | negativo | 0 | 1 |
| `a == b`, mas com valores negativos envolvidos | 0 | 1 | 0 (ZF não depende do sinal dos operandos, só do resultado) |

> Note que esta tabela cobre apenas os casos "simples" (sem overflow). Quando há overflow signed (por exemplo, comparando o menor `int` possível com um valor positivo), SF sozinha pode enganar — é exatamente por isso que existe a flag OF, e por isso comparações signed corretas combinam SF **e** OF, não SF isoladamente. Isso será formalizado no Módulo 7 — por ora, memorize apenas que "SF indica sinal do resultado", sem assumir ainda que isso basta para decidir "maior ou menor" em todos os casos.

### Erro comum de interpretação

Achar que `cmp eax, ebx` altera `EAX` ou `EBX`. **Nenhum dos dois operandos é modificado** — `cmp` é, nesse sentido, "somente leitura" em relação aos seus operandos; o único efeito observável está nas flags.

## 7. `test` — teste bit a bit

### O que faz

De forma análoga a `cmp`, `test` executa uma operação **AND** entre os dois operandos, mas **descarta o resultado**, atualizando apenas as flags.

### Sintaxe geral

```asm
test operando1, operando2
```

Conceitualmente: calcula `operando1 AND operando2` (Parte 2, Seção 5.1), descarta o resultado, mas atualiza ZF (principalmente) com base nele.

### Uso mais comum: testar se um valor é zero

```asm
mov eax, 0
test eax, eax    ; calcula EAX AND EAX (ou seja, o próprio EAX), descartado
                   ; se EAX era 0, ZF = 1
```

`test eax, eax` é um idioma extremamente comum em Assembly gerado por compiladores, equivalente a perguntar "EAX é zero?" (o resultado de `X AND X` é o próprio `X`, então ZF reflete diretamente se `X` era zero). É a versão "lógica" do que `cmp eax, 0` faria de forma aritmética — ambas terminam checando ZF, mas `test` é preferida por ser uma instrução menor.

### Uso comum 2: testar bits específicos

```asm
mov al, 0b00000101   ; AL = 5
test al, 1             ; testa apenas o bit menos significativo
                        ; se esse bit for 1, ZF = 0; se for 0, ZF = 1
```

Isso corresponde ao padrão de C `if (valor & 1)` mencionado na Parte 2 (Seção 5.1) — mas aqui usado apenas para **testar** a condição, sem alterar `valor`.

### Exemplo comentado — `test` vs. `and`, lado a lado

Assim como fizemos na Parte 2 com as formas de `imul`, vale comparar diretamente `test` com `and` usando os mesmos operandos, para deixar a diferença de efeito bem concreta:

```asm
mov al, 0b00000110   ; AL = 6
and al, 0b00000010    ; calcula AL AND 2 = 2, e GRAVA o resultado em AL
                       ; AL agora vale 2 (foi sobrescrito!)
```

```asm
mov al, 0b00000110   ; AL = 6
test al, 0b00000010   ; calcula AL AND 2 = 2, mas DESCARTA o resultado
                       ; AL continua valendo 6 — só ZF foi atualizada (ZF = 0, pois o resultado não era zero)
```

Nos dois casos, o cálculo interno (`AL AND 2`) é idêntico. A diferença inteira está em **o que acontece com o resultado**: `and` grava por cima do destino; `test` descarta e só deixa rastro nas flags.

### Erro comum de interpretação

Confundir `test` com `and`. A diferença central: `and destino, origem` **sobrescreve** o destino com o resultado; `test operando1, operando2` **descarta** o resultado, afetando apenas as flags. Se você vir o destino sendo usado logo depois com seu valor original intacto, é sinal de que era `test`, não `and` — como no exemplo acima, onde `AL` permanece `6` após o `test`, mas viraria `2` após o `and` equivalente.

## 8. `cmp` vs. `test`: quando cada uma aparece

| | Operação interna | Uso típico |
|---|---|---|
| `cmp a, b` | `a - b` (descartado) | Comparar dois valores (igual? maior? menor?) |
| `test a, b` | `a AND b` (descartado) | Testar se é zero, ou se bits específicos estão ligados |

Ambas são "somente leitura" em relação aos operandos — a única coisa que muda após executá-las é RFLAGS.

## 9. Tabela-resumo da Parte 3

| Instrução | Efeito | Afeta o operando? | Preenchimento (shifts) |
|---|---|---|---|
| `shl d, n` | `d = d << n` | Sim | zeros pela direita |
| `shr d, n` | `d = d >> n` (unsigned) | Sim | zeros pela esquerda |
| `sar d, n` | `d = d >> n` (signed) | Sim | bit de sinal pela esquerda |
| `cmp a, b` | calcula `a - b`, descarta | **Não** | — |
| `test a, b` | calcula `a AND b`, descarta | **Não** | — |

## 10. Exemplo prático integrado

```asm
mov eax, 20
shr eax, 2        ; EAX = 20 / 4 = 5
cmp eax, 5         ; compara EAX com 5 → resultado da subtração (0) descartado, ZF = 1
test eax, eax      ; testa se EAX é zero → EAX AND EAX = 5 (não-zero), ZF = 0
```

Passo a passo:

1. `mov eax, 20` — EAX = 20.
2. `shr eax, 2` — desloca 2 bits à direita, equivalente a dividir por 4 (unsigned): EAX = 5.
3. `cmp eax, 5` — calcula `5 - 5 = 0` (descartado). Como o resultado foi zero, `ZF = 1`. EAX continua valendo 5 (cmp não altera operandos).
4. `test eax, eax` — calcula `5 AND 5 = 5` (descartado). Como o resultado não é zero, `ZF = 0`. Note que a flag ZF foi **sobrescrita** pela segunda instrução — cada instrução que afeta flags recalcula-as do zero, sem "lembrar" do estado anterior.

Esse último ponto é importante para o Módulo 7: **as flags sempre refletem apenas a última instrução que as afetou**, não um acúmulo histórico.

## 11. Exercícios

### Nível 1 — Interpretar uma instrução

1. Qual é a diferença fundamental entre `cmp` e `sub`, já que ambas calculam a mesma subtração?
2. Por que `sar` existe separadamente de `shr`, já que ambas deslocam bits para a direita?
3. O que `test eax, eax` verifica, na prática?
4. Qual é a diferença entre `and al, 2` e `test al, 2`, dado que ambas calculam a mesma operação AND internamente?

### Nível 2 — Interpretar algumas instruções

```asm
mov eax, 6
shl eax, 2
cmp eax, 24
```

5. Qual é o valor de `EAX` após o `shl`?
6. O que a flag ZF indica após o `cmp`, e por quê?

### Nível 3 — Acompanhar registradores e flags

```asm
mov al, 0b11110000   ; AL = -16 (signed, 8 bits)
sar al, 2
```

7. Qual é o valor final de `AL`, em decimal (signed)?

```asm
mov al, 0b11110000   ; AL = -16 (signed, 8 bits) / 240 (unsigned)
shr al, 2
```

8. Qual é o valor final de `AL` neste caso, em binário? Por que ele é diferente do resultado do exercício 7, mesmo partindo do mesmo valor e do mesmo deslocamento de 2 bits?

```asm
mov eax, 9
test eax, 1
```

9. O que aconteceria com ZF nesse trecho, e por quê? (Dica: pense em 9 em binário e no bit menos significativo.)

---

## 12. Respostas

1. `sub destino, origem` **grava** o resultado da subtração no destino, sobrescrevendo-o. `cmp operando1, operando2` calcula a mesma subtração, mas **descarta** o resultado — o único efeito observável é a atualização das flags. Os operandos de `cmp` permanecem inalterados.
2. Porque `shr` sempre preenche com zeros, o que "destrói" o sinal de um valor negativo (o resultado passa a ser interpretado como um número positivo grande). `sar` preenche repetindo o bit de sinal original, preservando corretamente o comportamento de uma divisão signed por potência de 2. Sem `sar`, não haveria como deslocar bits à direita mantendo o sinal correto para valores negativos.
3. Verifica se `EAX` é igual a zero. Como `test eax, eax` calcula `EAX AND EAX` (que é o próprio EAX) e descarta o resultado, a flag ZF reflete diretamente se EAX era zero (ZF=1) ou não (ZF=0).
4. `and al, 2` calcula `AL AND 2` e **grava** o resultado de volta em `AL`, sobrescrevendo seu valor original. `test al, 2` calcula exatamente a mesma operação `AL AND 2`, mas **descarta** o resultado — `AL` permanece com seu valor original, e o único efeito visível é a atualização de ZF (e outras flags) com base no que o resultado teria sido.
5. `mov eax, 6` → EAX = 6. `shl eax, 2` desloca 2 bits à esquerda, equivalente a multiplicar por 4: EAX = 24.
6. `cmp eax, 24` calcula `24 - 24 = 0` (descartado). Como o resultado da subtração foi zero, `ZF = 1` — isso indica que os dois operandos comparados eram iguais.
7. `AL = 1111 0000` (-16 signed, 8 bits). `sar al, 2` desloca 2 posições à direita, preenchendo com o bit de sinal (1): resultado é `1111 1100`. Em decimal (signed), `1111 1100` = -4. (Confere com a divisão signed: -16 / 4 = -4.)
8. `AL = 1111 0000`. `shr al, 2` desloca 2 posições à direita, preenchendo com **zeros** (não com o bit de sinal): resultado é `0011 1100`. Isso é diferente do exercício 7 porque `shr` sempre preenche com `0` pela esquerda, independentemente do valor do bit de sinal original — enquanto `sar` preenche repetindo esse bit de sinal (`1`, nesse caso). Como o bit de sinal original era `1` (valor negativo), as duas instruções produzem resultados diferentes; se o valor original fosse positivo (bit de sinal `0`), ambas produziriam o mesmo resultado.
9. `9` em binário é `0000 1001`. O bit menos significativo é `1`. `test eax, 1` calcula `9 AND 1 = 1` (descartado). Como o resultado não é zero, `ZF = 0`. Isso corresponde a "9 é um número ímpar" (o bit menos significativo ligado indica valor ímpar).

---

*Módulo anterior: [Módulo 6 — Instruções Fundamentais (Parte 2)](./06-instrucoes-fundamentais-parte2.md)*
*Próximo módulo: [Módulo 7 — Flags](./07-flags.md)*
