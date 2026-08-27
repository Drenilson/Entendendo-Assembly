# Módulo 6 — Instruções Fundamentais (Parte 2)

> **Arquitetura:** x86-64
> **Sistema operacional de referência:** Linux
> **ABI:** System V AMD64
> **Sintaxe:** Intel

## 1. Introdução

Continuando a Parte 1, veremos agora: `imul`, `idiv`, `inc`, `dec`, e o grupo de instruções lógicas `xor`, `and`, `or`, `not`.

Este é, provavelmente, o conjunto de instruções mais denso do curso até aqui — envolve multiplicação e divisão signed (com seus registradores implícitos), e operações bit a bit que exigem pensar em binário, não em decimal. Por isso, vamos manter o mesmo formato de apresentação usado na Parte 1 (o que faz, sintaxe, o que muda, exemplo comentado, erro comum), mas com **mais exemplos por instrução** e, para as operações lógicas, sempre acompanhados de diagramas bit a bit — é a forma mais confiável de internalizar o que está de fato acontecendo, em vez de decorar regras soltas.

> **Dica antes de começar:** tenha a tabela de conversão hex↔binário do Módulo 4 (Seção 4.1) por perto enquanto lê esta parte. Praticamente todo exemplo aqui exige visualizar valores em binário, e ir convertendo de cabeça só vai atrapalhar seu raciocínio no começo.

## 2. `imul` — multiplicação (signed)

### O que faz

Multiplica dois valores. O nome vem de *integer multiply*, e o "i" indica que trata os operandos como **signed** (com sinal) — existe uma contraparte `mul` para valores unsigned, que não veremos em detalhe agora por ser menos comum em código gerado por compiladores C modernos.

Diferente de `add`/`sub`, que sempre têm exatamente 2 operandos, `imul` tem **três formas** diferentes de uso — e essa flexibilidade é justamente a fonte de confusão mais comum ao lê-la pela primeira vez.

### Sintaxe geral

```asm
imul destino, origem              ; destino = destino * origem
imul destino, origem, imediato    ; destino = origem * imediato
imul origem                       ; RDX:RAX = RAX * origem (forma de 1 operando, menos comum)
```

As duas primeiras são as mais comuns em código gerado por compiladores. A terceira (1 operando) segue uma lógica parecida com a de `idiv`, que veremos na Seção 3 — vale só reconhecê-la por ora; voltaremos a essa relação depois.

### O que muda

O destino recebe o resultado da multiplicação. Assim como `add`/`sub`, flags também são afetadas (com destaque para CF e OF, que sinalizam se o resultado "não coube" no tamanho do destino — ou seja, se o valor real da multiplicação era maior do que cabe em 32 ou 64 bits, por exemplo).

### Exemplo comentado 1 — forma de 2 operandos

```asm
mov eax, 6
imul eax, 7        ; EAX = EAX * 7 → EAX = 42
```

Aqui, `EAX` participa dos dois lados da operação: é lido (valor 6), multiplicado por 7, e o resultado (42) é escrito de volta nele mesmo. É o padrão mais parecido com `add`/`sub` que você já conhece — "destino = destino OP origem".

### Exemplo comentado 2 — forma de 3 operandos

```asm
mov ebx, 3
imul eax, ebx, 10  ; EAX = EBX * 10 → EAX = 30 (EBX não muda)
```

Aqui a lógica muda: o valor **anterior** de `EAX` é completamente ignorado — nem chega a ser lido. O que acontece é `EBX * 10`, e o resultado vai para `EAX`. É como se `EAX` fosse "só a caixa de saída", sem participar do cálculo.

### Exemplo comentado 3 — comparando as duas formas lado a lado

Para deixar a diferença ainda mais clara, veja o que acontece se você já tiver um valor em `EAX` antes de cada uma:

```asm
mov eax, 100      ; EAX = 100 (valor "antigo", antes da multiplicação)
mov ebx, 3

imul eax, ebx      ; forma de 2 operandos: EAX = EAX * EBX = 100 * 3 = 300
```

```asm
mov eax, 100      ; EAX = 100 (valor "antigo")
mov ebx, 3

imul eax, ebx, 10  ; forma de 3 operandos: EAX = EBX * 10 = 3 * 10 = 30
                    ; o "100" que estava em EAX é simplesmente descartado
```

Note o resultado: na primeira, o 100 antigo de `EAX` **participa** do cálculo (é multiplicado por EBX). Na segunda, o 100 antigo é **irrelevante** — só importa `EBX * 10`. Essa é exatamente a armadilha de leitura mais comum com `imul`.

### Exemplo comentado 4 — com valor negativo (por que o "i" de signed importa)

```asm
mov eax, -4
imul eax, 5        ; EAX = -4 * 5 = -20
```

Como `imul` interpreta os operandos como signed, o resultado é corretamente `-20`. Se a mesma sequência de bits fosse interpretada como unsigned (usando `mul` em vez de `imul`), o resultado numérico seria completamente diferente, porque o valor de partida (`-4` em complemento de dois) representaria, sem sinal, um número muito grande. Isso conecta diretamente com o que vimos no Módulo 4, Seção 5 — a mesma sequência de bits, duas interpretações possíveis.

### `imul origem` — a forma de 1 operando

Essa forma existe para o mesmo problema que `idiv` resolve: às vezes o resultado de uma multiplicação **não cabe** no tamanho do registrador de origem. Multiplicar dois números de 32 bits pode gerar um resultado de até 64 bits — por isso, assim como `idiv`, essa forma de `imul` usa um **par de registradores** para guardar o resultado completo, em vez de um único registrador.

**Sintaxe:**

```asm
imul origem       ; RDX:RAX = RAX * origem  (em 32 bits: EDX:EAX = EAX * origem)
```

O **multiplicando** é sempre implícito: `RAX` (ou `EAX`, em 32 bits). O `origem` que você escreve é o **multiplicador**. O resultado, por sua vez, não vai para um único registrador — ele é dividido entre dois:

- A parte **baixa** (os bits menos significativos do resultado) vai para `RAX`/`EAX`.
- A parte **alta** (os bits mais significativos, o "excedente" que não coube em 32/64 bits) vai para `RDX`/`EDX`.

**Exemplo comentado — resultado que cabe inteiro em EAX:**

```asm
mov eax, 6
mov ebx, 7
imul ebx          ; EDX:EAX = EAX * EBX = 6 * 7 = 42
                   ; EAX = 42, EDX = 0
```

Como `42` é um número pequeno, ele cabe inteiro em `EAX`, e `EDX` fica em `0` — não há "excedente" para guardar ali.

**Exemplo comentado — resultado que estoura o tamanho de EAX:**

```asm
mov eax, 0xFFFFFFFF   ; EAX = maior valor unsigned de 32 bits (4294967295)
mov ebx, 2
imul ebx               ; EDX:EAX = EAX * EBX = 4294967295 * 2 = 8589934590
```

O resultado real (`8589934590`) é maior do que cabe em 32 bits (o máximo é `4294967295`). Por isso, ele é dividido entre os dois registradores: `EDX` recebe a parte alta e `EAX` a parte baixa — juntos, formando o valor de 64 bits completo. (Não se preocupe em calcular os valores exatos de cabeça aqui — o ponto é entender *que* o resultado é repartido, e não que ele "estoura e é perdido" como aconteceria com a forma de 2 operandos.)

**Por que isso é o "espelho" de `idiv`:**

Vale a comparação direta, porque ajuda a fixar as duas instruções juntas:

| | Operandos explícitos | Implícito (par de registradores) | O que o par representa |
|---|---|---|---|
| `imul origem` | 1 (o multiplicador) | `RDX:RAX` | **Saída**: resultado completo da multiplicação |
| `idiv origem` | 1 (o divisor) | `RDX:RAX` | **Entrada**: dividendo completo a ser dividido |

Em `imul` (forma de 1 operando), o par `RDX:RAX` é onde o resultado **sai**. Em `idiv`, o par `RDX:RAX` é de onde o dividendo **entra**. É basicamente a mesma ideia de "um valor pode ser grande demais para um único registrador", só que aplicada em direções opostas — multiplicação pode *gerar* um valor grande; divisão pode *precisar* de um valor grande como entrada.

### Erro comum de interpretação

Confundir a forma de dois operandos (`imul destino, origem`, que multiplica o destino por algo, **usando** o valor atual do destino) com a de três operandos (`imul destino, origem, imediato`, que multiplica **origem** por um valor imediato e guarda em destino — o valor original de destino nem é lido). São padrões parecidos visualmente, mas com semânticas diferentes — sempre conte quantos operandos a instrução tem antes de interpretar, e pergunte "o destino participa do cálculo, ou só recebe o resultado?". Já na forma de 1 operando, o erro comum é esperar que o resultado esteja inteiramente em `EAX`/`RAX`, como acontece nas outras duas formas — aqui, sempre cheque `EDX`/`RDX` também, para não "perder" a parte mais significativa do resultado ao ler ou depurar código.

## 3. `idiv` — divisão (signed)

### O que faz

Divide um número por outro, também tratando os operandos como **signed**. É a instrução mais "diferente" desta seção, porque **não segue o padrão `destino, origem`** — ela sempre opera sobre um par fixo de registradores, e produz **dois resultados** ao mesmo tempo (quociente e resto), não apenas um.

### Sintaxe geral

```asm
idiv divisor
```

Onde `divisor` é um único operando (registrador ou memória). O **dividendo** é implícito: para uma divisão de 32 bits, é o par `EDX:EAX` (EDX contém a parte alta, EAX a parte baixa, formando um valor de 64 bits); para 64 bits, é o par `RDX:RAX`.

### Por que um "par" de registradores, e não só um?

Vale parar aqui e entender o motivo, porque é uma decisão de design que confunde bastante gente. A CPU permite dividir um número **maior** do que caberia em um único registrador — por isso o dividendo é formado por dois registradores concatenados (`EDX:EAX` funcionando como um único valor de 64 bits, onde EDX guarda os 32 bits mais significativos e EAX os 32 bits menos significativos). Na grande maioria dos casos em código gerado por C, você não está de fato dividindo um número gigante — mas o hardware exige esse formato de qualquer forma, então é preciso **preparar EDX corretamente** antes de dividir, mesmo que seu dividendo "lógico" caiba inteiro em EAX.

### O que muda

Após a execução:

- O **quociente** é colocado em `EAX` (ou `RAX`, em 64 bits).
- O **resto** é colocado em `EDX` (ou `RDX`).

### Exemplo comentado 1 — divisão simples, com preparação via `cdq`

```asm
mov eax, 17
cdq              ; estende o sinal de EAX para EDX:EAX (prepara EDX corretamente)
mov ebx, 5
idiv ebx         ; EDX:EAX / EBX → EAX = 3 (quociente), EDX = 2 (resto)
```

> **Nota: o que exatamente `cdq` faz.** `cdq` (*Convert Doubleword to Quadword*) é uma instrução auxiliar que "estende o sinal" de `EAX` para `EDX`. Concretamente: ela olha para o bit mais significativo de `EAX` (o bit de sinal, Módulo 4 Seção 5) e copia esse mesmo bit para **todos** os 32 bits de `EDX`. Se `EAX` for positivo (bit de sinal = 0), `EDX` vira `0x00000000`. Se `EAX` for negativo (bit de sinal = 1), `EDX` vira `0xFFFFFFFF`. Isso garante que o par `EDX:EAX`, lido como um único número de 64 bits, represente exatamente o mesmo valor que `EAX` representava sozinho em 32 bits — só que "esticado". Você vai vê-la quase sempre acompanhando `idiv` em código gerado por compiladores — por ora, reconheça o padrão `cdq` + `idiv` como "preparar e executar uma divisão signed".

### Exemplo comentado 2 — visualizando por que `cdq` é necessário

Veja o que aconteceria **sem** o `cdq`, para entender o problema que ele resolve:

```asm
mov eax, 17     ; EAX = 17
; (sem cdq aqui!)
mov ebx, 5
idiv ebx        ; PERIGO: EDX contém "lixo" de instruções anteriores
```

Se `EDX` já tivesse algum valor de uma operação anterior (e não `0`), a CPU trataria o dividendo como `EDX:EAX` inteiro — ou seja, um número gigante e incorreto, não os "17" que você pretendia dividir. O resultado da divisão sairia completamente errado, ou a CPU poderia até gerar um erro de divisão (se o "número gigante" gerado por lixo em EDX não permitir um quociente válido no tamanho do registrador). É exatamente por isso que `cdq` (ou, em contextos unsigned, zerar EDX manualmente com `xor edx, edx` — veremos isso na Seção 5.3) sempre aparece antes de uma divisão.

### Exemplo comentado 3 — divisão com resto exato (resto = 0)

```asm
mov eax, 20
cdq
mov ebx, 4
idiv ebx         ; EAX = 5 (quociente), EDX = 0 (resto — divisão exata)
```

Aqui não há nada de especial na instrução em si, mas vale notar: quando a divisão é exata, o resto (`EDX`) simplesmente fica `0` — o comportamento não muda, só o resultado.

### Exemplo comentado 4 — divisão com dividendo negativo

```asm
mov eax, -17
cdq              ; como EAX é negativo, EDX vira 0xFFFFFFFF
mov ebx, 5
idiv ebx         ; EAX = -3 (quociente), EDX = -2 (resto)
```

Repare que, em divisão signed, o resto também pode ser negativo — isso é diferente da divisão inteira "matemática pura" que você talvez tenha aprendido na escola, e é o mesmo comportamento que a divisão inteira `/` produz em C quando um dos operandos é negativo (`-17 / 5 == -3` e `-17 % 5 == -2` em C, com truncamento em direção a zero).

### Erro comum de interpretação

Esquecer que `idiv` **não recebe o dividendo como operando** — ele já é fixo (`EDX:EAX`). Ao ler `idiv ebx`, não pense "EAX dividido por EBX" isoladamente — pense "o par EDX:EAX dividido por EBX", mesmo que, na prática (com EDX corretamente zerado/estendido), o efeito pareça só "EAX / EBX". Um segundo erro comum é esquecer o `cdq` (ou equivalente) antes de `idiv` e assumir que `EDX` "já estava zerado" — nunca assuma isso; sempre prepare `EDX` explicitamente antes de dividir.

## 4. `inc` e `dec` — incrementar e decrementar

### O que fazem

`inc` soma 1 ao operando; `dec` subtrai 1. São formas mais compactas de `add destino, 1` e `sub destino, 1`.

### Sintaxe geral

```asm
inc destino
dec destino
```

### O que muda

O destino é incrementado/decrementado em 1. Flags são afetadas — com uma exceção notável: `inc`/`dec` **não afetam CF** (Carry Flag), diferente de `add`/`sub`. Isso é uma particularidade histórica de x86 que vale reter, pois pode importar ao ler código que depende de CF logo após um `inc`/`dec`.

### Exemplo comentado 1 — uso básico

```asm
mov eax, 9
inc eax        ; EAX = 10
dec eax        ; EAX = 9
```

### Exemplo comentado 2 — em um contexto mais realista: contador de laço

Esse par de instruções aparece constantemente em laços (`for`, `while`) compilados de C. Por exemplo, o C:

```c
int contador = 0;
contador++;   // ou contador = contador + 1
```

tende a virar, em Assembly:

```asm
mov dword [rbp-4], 0     ; contador = 0 (armazenado na stack, Módulo 5)
inc dword [rbp-4]         ; contador++
```

Note que `inc` também pode operar diretamente sobre um endereço de memória (com o sufixo `dword` explícito, Módulo 4 Seção 2, já que o tamanho não pode ser deduzido de um registrador aqui) — não precisa ser sempre um registrador.

### Exemplo comentado 3 — comparando `inc` com `add equivalente`

```asm
mov eax, 5
inc eax          ; EAX = 6  (equivalente a add eax, 1)

mov ebx, 5
add ebx, 1       ; EBX = 6  (mesmo resultado numérico)
```

Numericamente, o resultado é idêntico. A diferença real está só nas flags (CF não é tocado por `inc`) e, historicamente, no tamanho em bytes da instrução — outro motivo pelo qual compiladores preferem `inc`/`dec` quando o valor somado/subtraído é exatamente 1.

### Erro comum de interpretação

Achar que `inc`/`dec` são "iguais" a `add`/`sub` em todos os aspectos, incluindo flags. A diferença no tratamento de CF é sutil, mas real — e importa especificamente em código que testa CF logo depois (algo raro em C compilado comum, mas comum em Assembly escrito à mão para controle fino de loops com carry).

## 5. Operações lógicas bit a bit: `and`, `or`, `xor`, `not`

Essas quatro instruções operam **bit a bit** — cada bit do resultado depende apenas dos bits correspondentes dos operandos, sem "carregar" nada entre posições (diferente de somas e subtrações, onde um "vai-um" de uma posição pode afetar a posição seguinte). Isso é uma diferença fundamental: em `and`/`or`/`xor`/`not`, cada posição de bit é **completamente independente** das demais.

Essas operações têm uma correspondência direta com operadores de C: `&` (and), `|` (or), `^` (xor), `~` (not) — os chamados *operadores bitwise* de C, diferentes de `&&`, `||` (que operam sobre valores lógicos verdadeiro/falso inteiros, não bit a bit).

### 5.1 `and` — E lógico

Cada bit do resultado é 1 apenas se **ambos** os bits correspondentes forem 1.

```asm
and destino, origem     ; destino = destino AND origem
```

```
  1010
& 1100
------
  1000
```

**Tabela-verdade do AND, bit a bit:**

| Bit A | Bit B | A AND B |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |

**Uso comum 1 — mascarar bits:** isolar apenas alguns bits de um valor, zerando o resto.

```asm
mov al, 0xF3      ; AL = 1111 0011
and al, 0x0F      ; máscara: 0000 1111
                   ; resultado: 0000 0011 = 0x03
```

Repare o padrão: onde a máscara tem `0`, o resultado é forçosamente `0` (independente do valor original). Onde a máscara tem `1`, o bit original "passa" inalterado. É por isso que dizemos que `and` com uma máscara "preserva alguns bits e zera o resto".

**Uso comum 2 — testar se um bit específico está ligado**, sem alterar o valor original (frequentemente combinado com uma instrução de comparação, que veremos no Módulo 7):

```asm
mov al, 0x05       ; AL = 0000 0101
and al, 0x01       ; isola apenas o bit menos significativo → 0000 0001 = 1
                    ; (se o resultado for 0, o bit 0 original era 0; se for 1, era 1)
```

Essa técnica corresponde diretamente ao C `if (valor & 1)` para testar se um número é ímpar.

### 5.2 `or` — OU lógico

Cada bit do resultado é 1 se **pelo menos um** dos bits correspondentes for 1.

```asm
or destino, origem      ; destino = destino OR origem
```

```
  1010
| 1100
------
  1110
```

**Tabela-verdade do OR, bit a bit:**

| Bit A | Bit B | A OR B |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 1 |

**Uso comum — "ligar" bits específicos, sem afetar os demais:**

```asm
mov al, 0x01       ; AL = 0000 0001
or al, 0x08         ; máscara: 0000 1000
                     ; resultado: 0000 1001 = 0x09
```

Note a diferença de comportamento em relação a `and`: aqui, onde a máscara tem `1`, o resultado é forçosamente `1` (independente do valor original); onde a máscara tem `0`, o bit original "passa" inalterado. `and` zera seletivamente; `or` liga seletivamente — são operações "espelhadas" uma da outra.

### 5.3 `xor` — OU exclusivo

Cada bit do resultado é 1 se os bits correspondentes forem **diferentes** entre si (exatamente um dos dois é 1).

```asm
xor destino, origem     ; destino = destino XOR origem
```

```
  1010
^ 1100
------
  0110
```

**Tabela-verdade do XOR, bit a bit:**

| Bit A | Bit B | A XOR B |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

**Uso comum e muito frequente em Assembly gerado por compiladores:** `xor eax, eax` é a forma idiomática de **zerar um registrador**. Qualquer valor "XOR" consigo mesmo resulta em zero — olhando a tabela-verdade acima, isso faz sentido: cada bit está sendo comparado com uma cópia idêntica de si mesmo, então nunca são "diferentes" (a condição que geraria 1). Por que usar isso em vez de `mov eax, 0`? Historicamente, `xor` é uma instrução menor em bytes e, em algumas CPUs, mais rápida — o compilador prefere essa forma por otimização, mesmo que o efeito final seja idêntico a `mov eax, 0`.

```asm
xor eax, eax      ; EAX = 0 (idioma comum — leia como "zere EAX")
```

> Isso é importante para leitura: sempre que você ver `xor reg, reg` **com o mesmo registrador dos dois lados**, reconheça isso instantaneamente como "zerar aquele registrador", não como uma operação lógica "de verdade" a ser calculada. Esse mesmo idioma também explica por que vimos, na Seção 3 deste módulo, uma alternativa a `cdq`: `xor edx, edx` é uma forma comum de zerar `EDX` manualmente antes de uma divisão **unsigned** (onde não se quer extensão de sinal, apenas zero puro).

**Um segundo uso comum de `xor`: alternar (toggle) bits.**

```asm
mov al, 0x0F       ; AL = 0000 1111
xor al, 0xFF        ; XOR com todos os bits em 1
                     ; resultado: 1111 0000 = 0xF0
```

Fazer XOR de qualquer valor com uma sequência de todos os bits `1` **inverte** esse valor por completo — o que nos leva diretamente à próxima instrução.

### 5.4 `not` — negação bit a bit (complemento de um)

Diferente das três anteriores, `not` tem **um único operando**: inverte cada bit (0 vira 1, 1 vira 0).

```asm
not destino      ; destino = NOT destino (inverte todos os bits)
```

```
NOT 1010
--------
    0101
```

**Tabela-verdade do NOT, bit a bit:**

| Bit A | NOT A |
|---|---|
| 0 | 1 |
| 1 | 0 |

**Exemplo comentado — `not` em um byte inteiro:**

```asm
mov al, 0x3C       ; AL = 0011 1100
not al              ; AL = 1100 0011 = 0xC3
```

> **Atenção para não confundir:** `not` (inverter bits) é diferente de negar aritmeticamente um número (transformar um valor positivo no seu equivalente negativo). Para negação aritmética em complemento de dois, existe a instrução `neg` (que inverte os bits **e** soma 1 — exatamente o processo que vimos no Módulo 4, Seção 5). `not` sozinho é apenas o "inverter bits", sem o `+1`.

**Comparando `not` e `neg` lado a lado, com o mesmo valor de entrada:**

```asm
mov al, 5          ; AL = 0000 0101

; not al daria:      1111 1010  = 250 (unsigned) ou -6 (signed)
; neg al daria:       1111 1011  = 251 (unsigned) ou -5 (signed)
```

Repare que `not` e `neg` produzem resultados **diferentes por exatamente 1** — o que faz sentido, já que `neg` é literalmente "`not`, e depois soma 1" (o algoritmo de complemento de dois do Módulo 4, Seção 5, aplicado como instrução).

### O que muda (para todo o grupo `and`/`or`/`xor`/`not`)

O destino é sobrescrito com o resultado da operação bit a bit. Flags são afetadas — mas, diferente de `add`/`sub`, essas instruções sempre **zeram CF e OF** (não há conceito de "carry" ou "overflow aritmético" em operações puramente lógicas, já que cada bit é processado de forma independente, sem propagação entre posições).

## 6. Tabela-resumo da Parte 2

| Instrução | Efeito | Operandos | Observação |
|---|---|---|---|
| `imul d, o` | `d = d * o` (signed) | 2 (ou 3, com imediato) | Forma de 1 operando: `RDX:RAX = RAX * o` |
| `idiv o` | `EDX:EAX / o` → quociente em EAX, resto em EDX | 1 | Dividendo é implícito; requer `cdq` (ou equivalente) antes |
| `inc d` | `d = d + 1` | 1 | Não afeta CF |
| `dec d` | `d = d - 1` | 1 | Não afeta CF |
| `and d, o` | `d = d AND o` (bit a bit) | 2 | Usado para mascarar bits |
| `or d, o` | `d = d OR o` (bit a bit) | 2 | Usado para ligar bits |
| `xor d, o` | `d = d XOR o` (bit a bit) | 2 | `xor reg, reg` = idioma para zerar |
| `not d` | `d = NOT d` (inverte bits) | 1 | Diferente de negação aritmética (`neg`) |

## 7. Exemplo prático integrado

```asm
xor eax, eax      ; EAX = 0 (idioma: zerar)
mov eax, 8
imul eax, 3       ; EAX = 8 * 3 = 24
inc eax           ; EAX = 25
mov ebx, 0x0F
and eax, ebx      ; EAX = 25 AND 0x0F
```

Passo a passo:

1. `xor eax, eax` — EAX é zerado (embora, logo em seguida, seja sobrescrito — isso ilustra que nem sempre esse idioma "sobrevive" até o fim do trecho, mas é útil reconhecê-lo mesmo assim).
2. `mov eax, 8` — EAX passa a valer 8.
3. `imul eax, 3` — EAX = 8 × 3 = 24.
4. `inc eax` — EAX = 25.
5. `mov ebx, 0x0F` — EBX = 15 (`0000 1111` em binário).
6. `and eax, ebx` — 25 em binário é `0001 1001`. Fazendo AND bit a bit com `0000 1111`: resultado é `0000 1001` = 9. EAX passa a valer **9**.

## 7.1 Segundo exemplo integrado — envolvendo `idiv`

Para fechar com um exemplo que também usa divisão, considere este trecho, que simula, em Assembly, algo como `int resultado = (17 * 2) / 3;` em C:

```asm
mov eax, 17
imul eax, 2       ; EAX = 17 * 2 = 34
cdq                ; prepara EDX:EAX para divisão signed
mov ebx, 3
idiv ebx           ; EDX:EAX / EBX → EAX = quociente, EDX = resto
```

Passo a passo:

1. `mov eax, 17` — EAX = 17.
2. `imul eax, 2` — EAX = 34.
3. `cdq` — como EAX (34) é positivo, EDX é preenchido com `0x00000000`.
4. `mov ebx, 3` — EBX = 3.
5. `idiv ebx` — divide o par `EDX:EAX` (que representa 34) por 3: quociente = 11, resto = 1. `EAX` passa a valer **11**, `EDX` passa a valer **1**.

Isso corresponde exatamente a `34 / 3 == 11` e `34 % 3 == 1` em C (divisão inteira, truncada).

## 8. Exercícios

### Nível 1 — Interpretar uma instrução

1. O que faz `xor edx, edx`, e por que esse padrão é reconhecido como um idioma especial?
2. Qual é a diferença entre `not eax` e `neg eax`?
3. Por que `idiv` não segue o padrão `destino, origem` das outras instruções aritméticas vistas até aqui?
4. Qual é a diferença entre a forma de 2 operandos e a forma de 3 operandos de `imul`?
5. Por que `cdq` costuma aparecer logo antes de `idiv`?
5.1. Em que sentido a forma de 1 operando de `imul` é "o espelho" de `idiv`?

### Nível 2 — Interpretar algumas instruções

```asm
mov eax, 4
imul eax, 5
dec eax
mov ebx, 0xFF
and eax, ebx
```

6. Qual é o valor final de `EAX`?

```asm
mov al, 0x5A
or al, 0x0F
and al, 0xF0
```

7. Qual é o valor final de `AL`, em hexadecimal? (Dica: converta cada passo para binário.)

### Nível 3 — Acompanhar registradores (com idiv)

```asm
mov eax, 23
cdq
mov ecx, 4
idiv ecx
```

8. Qual é o valor final de `EAX`?
9. Qual é o valor final de `EDX`?

```asm
mov eax, -30
cdq
mov ebx, 7
idiv ebx
```

10. Qual é o valor final de `EAX` (quociente)? E de `EDX` (resto)?

### Nível 4 — Combinando tudo

```asm
mov eax, 3
imul eax, eax, 3   ; forma de 3 operandos
inc eax
xor ebx, ebx
mov ebx, 0x1F
and eax, ebx
```

11. Percorra cada instrução e determine o valor final de `EAX`.

---

## 9. Respostas

1. Zera o registrador `EDX` (qualquer valor XOR consigo mesmo é zero, como mostra a tabela-verdade do XOR). É reconhecido como idioma porque compiladores usam essa forma, em vez de `mov edx, 0`, por questões de tamanho/desempenho da instrução — ao ler Assembly, `xor reg, reg` deve ser lido diretamente como "zere este registrador".
2. `not eax` inverte todos os bits de `EAX` (complemento de um), sem qualquer relação direta com o valor numérico "negativo equivalente". `neg eax` calcula o negativo aritmético de `EAX` em complemento de dois (inverte os bits **e** soma 1) — é a operação que efetivamente transforma, por exemplo, 5 em -5. Os dois resultados diferem sempre em exatamente 1.
3. Porque `idiv` opera sobre um **par fixo de registradores** (EDX:EAX ou RDX:RAX) como dividendo implícito, em vez de usar dois operandos explícitos como `add`/`sub`/`imul` (na forma de 2 operandos). Isso existe porque a divisão pode gerar um quociente e um resto simultaneamente, exigindo dois "lugares" de saída (EAX para quociente, EDX para resto), além de permitir dividendos maiores que o próprio registrador de origem.
4. Na forma de 2 operandos (`imul destino, origem`), o valor **atual** de destino participa do cálculo: `destino = destino * origem`. Na forma de 3 operandos (`imul destino, origem, imediato`), o valor atual de destino é ignorado — ele só recebe o resultado de `origem * imediato`, sem nunca ser lido.
5. Porque `idiv` sempre trabalha sobre o par `EDX:EAX` como dividendo, mesmo quando o valor "lógico" que você quer dividir cabe inteiro em EAX. Se `EDX` não for preparado corretamente antes (com `cdq`, que estende o bit de sinal de EAX para todo o EDX), ele pode conter lixo de instruções anteriores, fazendo a CPU interpretar um dividendo completamente errado.
5.1. Porque as duas usam o mesmo par de registradores (`RDX:RAX`) para lidar com valores maiores do que cabem em um único registrador — só que em direções opostas. Em `imul` (1 operando), o par é onde o **resultado** (que pode ser grande demais para RAX sozinho) **sai**. Em `idiv`, o par é de onde o **dividendo** (que pode ser grande demais para representar em RAX sozinho) **entra**.
6. `mov eax, 4` → EAX = 4. `imul eax, 5` → EAX = 20. `dec eax` → EAX = 19. `mov ebx, 0xFF` → EBX = 255. `and eax, ebx` → 19 AND 255 = 19 (pois 19 em binário, `0001 0011`, já "cabe" inteiramente dentro da máscara `1111 1111`, então nada é zerado). Valor final de `EAX`: **19**.
7. `AL = 0x5A` (`0101 1010`). `or al, 0x0F` (`0000 1111`) → `0101 1111` = `0x5F`. `and al, 0xF0` (`1111 0000`) → `0101 0000` = `0x50`. Valor final de `AL`: **0x50**.
8. `EAX = 23`, `cdq` estende o sinal (EDX = 0, pois 23 é positivo). `ECX = 4`. `idiv ecx` calcula `23 / 4`: quociente = 5, resto = 3. Valor final de `EAX` (quociente): **5**.
9. Valor final de `EDX` (resto): **3**.
10. `EAX = -30` (negativo, então `cdq` preenche EDX com `0xFFFFFFFF`). `EBX = 7`. `idiv ebx` calcula `-30 / 7`: em divisão signed truncada em direção a zero, o quociente é **-4** e o resto é **-2** (confira: `-4 * 7 = -28`, e `-28 + (-2) = -30`, batendo com o dividendo original).
11. `mov eax, 3` → EAX = 3 (será descartado no próximo passo). `imul eax, eax, 3` (forma de 3 operandos) → EAX = EAX(3) * 3 = 9. `inc eax` → EAX = 10. `xor ebx, ebx` → EBX = 0 (mas será sobrescrito a seguir). `mov ebx, 0x1F` → EBX = 31 (`0001 1111`). `and eax, ebx` → 10 em binário é `0000 1010`; AND com `0001 1111` resulta em `0000 1010` = 10 (nada muda, pois todos os bits de 10 já "cabem" na máscara). Valor final de `EAX`: **10**.

---

*Módulo anterior: [Módulo 6 — Instruções Fundamentais (Parte 1)](./06-instrucoes-fundamentais-parte1.md)*
*Próximo: [Módulo 6 — Instruções Fundamentais (Parte 3)](./06-instrucoes-fundamentais-parte3.md)*
