# Módulo 4 — Dados

> **Arquitetura:** x86-64
> **Sistema operacional de referência:** Linux
> **ABI:** System V AMD64
> **Sintaxe:** Intel

## 1. Introdução

Nos módulos anteriores, vimos *onde* os dados podem estar (registradores, memória) e *como* a CPU processa instruções. Mas ainda não paramos para entender **o que é um dado**, do ponto de vista da CPU.

Este módulo responde a uma pergunta simples, porém fundamental: **quando a CPU olha para um monte de bits, como ela sabe o que eles significam?**

A resposta curta é: **ela não sabe, a menos que a instrução diga**. Bits, por si só, não têm significado — o significado vem do **tamanho** que a instrução assume e da **operação** que está sendo feita. Este módulo constrói essa ideia com calma.

## 2. Bits e bytes

- **Bit**: a menor unidade de informação, com valor 0 ou 1.
- **Byte**: um agrupamento de 8 bits. É a menor unidade de memória endereçável em x86-64 — ou seja, cada endereço de memória aponta para exatamente 1 byte (veremos isso em detalhe no Módulo 5).

A partir do byte, x86-64 define nomes padronizados para agrupamentos maiores:

| Nome | Tamanho | Bits |
|---|---|---|
| **Byte** | 1 byte | 8 bits |
| **Word** | 2 bytes | 16 bits |
| **Double word** (dword) | 4 bytes | 32 bits |
| **Quad word** (qword) | 8 bytes | 64 bits |

Esses nomes não são arbitrários — eles aparecem **diretamente** em código Assembly, especialmente quando o tamanho do dado não pode ser deduzido pelo contexto. Por exemplo:

```asm
mov byte [rax], 5      ; escreve 1 byte no endereço apontado por RAX
mov word [rax], 5      ; escreve 2 bytes (16 bits)
mov dword [rax], 5     ; escreve 4 bytes (32 bits)
mov qword [rax], 5     ; escreve 8 bytes (64 bits)
```

Repare a relação direta com o que vimos no Módulo 3: `AL` é do tamanho de um **byte**, `AX` do tamanho de uma **word**, `EAX` de uma **dword**, `RAX` de uma **qword**. Os nomes dos registradores e os nomes dos tamanhos de dado **não são coincidência** — eles compartilham a mesma origem histórica.

## 3. Por que o tamanho importa tanto?

Porque a mesma sequência de bits pode significar coisas completamente diferentes dependendo do tamanho considerado. Veja o valor em memória:

```
Endereço:   0x1000  0x1001  0x1002  0x1003
Byte:         11      22      33      44
```

Se uma instrução ler isso como **1 byte** a partir de `0x1000`, o valor é `0x11`. Se ler como **dword** (4 bytes) a partir de `0x1000`, o valor é `0x44332211` (note a ordem — veremos isso na Seção 6, *little-endian*). O mesmo conjunto de bytes na memória, lido de formas diferentes, produz valores diferentes.

**Conclusão prática:** ao ler Assembly, sempre pergunte "qual é o tamanho do dado envolvido nesta instrução?" — o registrador usado (`AL` vs `EAX` vs `RAX`) ou o sufixo explícito (`byte`, `word`, `dword`, `qword`) sempre indicam isso.

## 4. Hexadecimal: por que Assembly "fala" em hexadecimal

Números em Assembly aparecem quase sempre em **hexadecimal** (base 16), não em decimal. Isso não é estilo — é praticidade: cada dígito hexadecimal representa exatamente 4 bits, então **2 dígitos hexadecimais = exatamente 1 byte**.

```
Binário:      1111 1111
Hexadecimal:    F    F      →  0xFF
Decimal:              255
```

Em sintaxe Intel/NASM, hexadecimal costuma ser escrito com prefixo `0x` (ex.: `0xFF`) ou sufixo `h` (ex.: `0FFh`, dependendo da ferramenta). Este curso usará o formato `0x...`.

| Decimal | Binário | Hexadecimal |
|---|---|---|
| 0 | 0000 | 0x0 |
| 5 | 0101 | 0x5 |
| 10 | 1010 | 0xA |
| 15 | 1111 | 0xF |
| 255 | 1111 1111 | 0xFF |
| 4096 | 0001 0000 0000 0000 | 0x1000 |

Ao ler `mov eax, 0x2A`, você deve conseguir reconhecer rapidamente que `0x2A` equivale a 42 em decimal — mas, mais importante que a conversão exata, é entender que hexadecimal é apenas **outra forma de escrever o mesmo valor binário**, mais compacta e mais fácil de mapear para bytes.

## 4.1 Convertendo entre hexadecimal, decimal e binário

**Hexadecimal → Decimal**

Cada posição em hexadecimal vale uma potência de 16, da direita para a esquerda (16⁰, 16¹, 16², ...). Basta multiplicar cada dígito pelo seu peso posicional e somar.

```
0x2A = (2 × 16¹) + (A × 16⁰)
     = (2 × 16) + (10 × 1)
     = 32 + 10
     = 42
```

Outro exemplo, com 3 dígitos:

```
0x1F4 = (1 × 16²) + (F × 16¹) + (4 × 16⁰)
      = (1 × 256) + (15 × 16) + (4 × 1)
      = 256 + 240 + 4
      = 500
```

**Decimal → Hexadecimal**

Divide-se repetidamente por 16, guardando o resto de cada divisão. O hexadecimal é formado lendo os restos de baixo para cima.

```
42 ÷ 16 = 2  resto 10 (A)
 2 ÷ 16 = 0  resto 2

Lendo de baixo para cima: 2A  →  0x2A
```

**Hexadecimal ↔ Binário (o caminho mais rápido)**

Você raramente vai querer converter hex→decimal→binário passando por decimal — é mais trabalho que o necessário. Como cada dígito hex corresponde a exatamente 4 bits (Seção 4), a conversão é direta, dígito por dígito:

```
Hex:     2    A
Binário: 0010 1010
```

E o caminho inverso: agrupe o binário em blocos de 4 bits (a partir da direita) e converta cada bloco separadamente.

```
Binário: 1101 1110
Blocos:  1101 = D    1110 = E
Hex:     0xDE
```

Tabela de referência rápida (os 16 dígitos hex):

| Hex | Binário | Decimal | Hex | Binário | Decimal |
|---|---|---|---|---|---|
| 0 | 0000 | 0 | 8 | 1000 | 8 |
| 1 | 0001 | 1 | 9 | 1001 | 9 |
| 2 | 0010 | 2 | A | 1010 | 10 |
| 3 | 0011 | 3 | B | 1011 | 11 |
| 4 | 0100 | 4 | C | 1100 | 12 |
| 5 | 0101 | 5 | D | 1101 | 13 |
| 6 | 0110 | 6 | E | 1110 | 14 |
| 7 | 0111 | 7 | F | 1111 | 15 |

> Na prática, ao ler Assembly, o caminho hex↔binário é o que você vai usar o tempo todo (para visualizar flags, máscaras de bits, endereços). O caminho hex↔decimal é mais para entender "quanto vale esse número na vida real".

**Exercícios extras (Nível 1)**

10. Converta 0x7C para decimal usando o método posicional.
11. Converta 100 (decimal) para hexadecimal usando divisões sucessivas.
12. Converta 0xB6 diretamente para binário (sem passar por decimal).
13. Converta 1010 0011 (binário) diretamente para hexadecimal.

**Respostas**

10. 0x7C = (7×16) + (12×1) = 112 + 12 = 124
11. 100 ÷ 16 = 6 resto 4 → 0x64
12. 0xB6 = B(1011) 6(0110) → 1011 0110
13. 1010 0011 → A(1010) 3(0011) → 0xA3

## 5. Números negativos: complemento de dois

Registradores e posições de memória não têm um "sinal" embutido nos bits — eles são apenas sequências de 0s e 1s. A **interpretação** de um valor como positivo ou negativo depende inteiramente de como a instrução trata esses bits.

x86-64 (como praticamente toda arquitetura moderna) usa **complemento de dois** para representar inteiros negativos. A ideia central:

- O bit mais significativo funciona, na prática, como indicador de sinal (0 = não-negativo, 1 = negativo) **quando o valor é interpretado como signed**.
- Para obter o complemento de dois de um número, inverte-se todos os bits e soma-se 1.

Exemplo com um byte (8 bits):

```
5 em binário:        0000 0101

Inverter os bits:    1111 1010
Somar 1:             1111 1011   → isso é -5 em complemento de dois (8 bits)
```

Verificando: `1111 1011` interpretado como **unsigned** (sem sinal) vale 251. Interpretado como **signed** (com sinal, complemento de dois), vale -5. **O mesmo byte, duas interpretações possíveis.**

```
0000 0101  =  5   (mesmo padrão de bits)
1111 1011  =  251 (unsigned)  OU  -5 (signed)
```

> Isso é crucial: **os bits não "sabem" se são negativos ou não**. Quem decide isso é a instrução que os lê — algumas instruções (e desvios condicionais) tratam o dado como signed, outras como unsigned. Vamos aprofundar essa diferença no Módulo 7 (Flags), quando falarmos sobre comparações signed vs. unsigned.

### 5.1 Faixas de valores por tamanho

| Tamanho | Unsigned (sem sinal) | Signed (com sinal, complemento de dois) |
|---|---|---|
| Byte (8 bits) | 0 a 255 | -128 a 127 |
| Word (16 bits) | 0 a 65.535 | -32.768 a 32.767 |
| Dword (32 bits) | 0 a 4.294.967.295 | -2.147.483.648 a 2.147.483.647 |
| Qword (64 bits) | 0 a ~18,4 quintilhões | ~-9,2 a ~9,2 quintilhões |

Não é necessário decorar esses limites — o padrão importa mais que os números exatos: **metade da faixa unsigned vira a faixa negativa quando o valor é interpretado como signed**.

## 6. Little-endian: a ordem dos bytes na memória

x86-64 é uma arquitetura **little-endian**: ao armazenar um valor com mais de 1 byte na memória, o **byte menos significativo é armazenado no endereço mais baixo**.

Exemplo: armazenar o valor `0x12345678` (uma dword) a partir do endereço `0x1000`:

```
Endereço:    0x1000  0x1001  0x1002  0x1003
Byte:          78      56      34      12
```

Note que os bytes aparecem "invertidos" em relação à forma como escrevemos o número. Isso costuma confundir bastante quem está começando a examinar memória diretamente (por exemplo, em um depurador). A regra para lembrar:

> **Little-endian: o byte "menos importante" fica no endereço mais baixo.** ("Little" = a ponta pequena do número vem primeiro.)

Isso será revisitado no Módulo 5, quando lidarmos com leitura e escrita direta de memória.

## 7. Relação com tipos do C

Tipo em C é um conceito de linguagem de alto nível, usado porque o programador humano precisa pensar em termos de significado — "isso é um caractere", "isso é um contador", "isso é um endereço" — sem se preocupar, a cada linha, em quantos bytes aquilo ocupa ou como a CPU vai interpretar os bits. Em Assembly, esse significado já desapareceu: o que resta é apenas quantos bytes estão sendo manipulados e como essa operação específica os interpreta (signed/unsigned, por exemplo).

Tipo e tamanho são conceitos relacionados, mas não idênticos — tipo é uma abstração para humanos; tamanho é uma instrução para a CPU. O compilador é quem faz essa ponte, traduzindo a camada de significado (tipo) para a camada de mecânica pura (tamanho) que o processador entende.

Ainda assim, na prática (em Linux x86-64, com GCC), a correspondência de tamanhos costuma ser:

| Tipo em C | Tamanho típico | Nome em Assembly |
|---|---|---|
| `char` | 1 byte | byte |
| `short` | 2 bytes | word |
| `int` | 4 bytes | dword |
| `long` | 8 bytes | qword |
| `int*` (qualquer ponteiro) | 8 bytes | qword |

Um exemplo simples: o código C

```c
char  a = 5;
short b = 5;
int   c = 5;
long  d = 5;
```

tende a gerar Assembly que usa registradores/tamanhos diferentes para cada variável — algo como:

```asm
mov byte [rbp-1], 5    ; char a
mov word [rbp-4], 5    ; short b
mov dword [rbp-8], 5   ; int c
mov qword [rbp-16], 5  ; long d
```

(Não se preocupe com o significado exato de `[rbp-1]` agora — endereçamento com deslocamento é assunto do Módulo 5. O ponto aqui é apenas notar a correspondência de **tamanhos**.)

## 8. Exemplo prático integrado

```asm
mov al, 0xFF     ; AL = 0xFF (1 byte)
mov bx, 0xFF     ; BX = 0x00FF (2 bytes, o valor 0xFF cabe nos 8 bits baixos)
mov ecx, 0xFF    ; ECX = 0x000000FF (4 bytes)
```

Nas três instruções, o **valor lógico** é o mesmo (255, ou -1 se interpretado como signed em 1 byte), mas o **tamanho do dado manipulado** é diferente em cada linha — 1, 2 e 4 bytes, respectivamente. Isso é determinado pelo tamanho do registrador de destino escolhido em cada instrução.

Se `AL` (255 em unsigned) fosse interpretado como **signed**, valeria **-1**. Isso ilustra, de novo, a ideia central da Seção 5: **o bit padrão não muda — a interpretação é que depende do contexto.**

## 9. Exercícios

### Nível 1 — Conceitual

1. Qual é a diferença entre um "bit" e um "byte"?
2. Por que Assembly costuma usar hexadecimal em vez de decimal?
3. Explique, com suas palavras, por que a mesma sequência de bits pode representar valores diferentes dependendo de ser interpretada como signed ou unsigned.
4. O que significa dizer que x86-64 é "little-endian"?
5. Por que "tipo em C" e "tamanho em Assembly" não são a mesma coisa, mesmo estando relacionados?

### Nível 2 — Aplicação

6. Converta `0x2A` para decimal e para binário (8 bits).
7. O byte `1000 0000` (0x80) representa qual valor em decimal se interpretado como **unsigned**? E se interpretado como **signed** (complemento de dois, 8 bits)?
8. O valor `0xDEADBEEF` (uma dword) é armazenado a partir do endereço `0x2000` em um sistema little-endian. Quais bytes ficam nos endereços `0x2000`, `0x2001`, `0x2002` e `0x2003`?
9. Para o código C abaixo, indique qual "nome de tamanho" Assembly (byte/word/dword/qword) provavelmente corresponde a cada variável:

```c
char  x;
long  y;
int   z;
short w;
```

---

## 10. Respostas

1. Um **bit** é a menor unidade de informação (0 ou 1). Um **byte** é um agrupamento de 8 bits, e é a menor unidade endereçável na memória em x86-64.
2. Porque cada dígito hexadecimal corresponde exatamente a 4 bits, então 2 dígitos hexadecimais representam exatamente 1 byte — isso torna muito mais fácil visualizar e converter valores binários do que usar decimal.
3. Os bits armazenados não mudam — o que muda é a **interpretação** aplicada a eles. Em complemento de dois, se o valor for tratado como signed, o bit mais significativo passa a indicar sinal e os valores "altos" (na leitura unsigned) correspondem a valores negativos. A mesma sequência de bits pode, portanto, representar um número positivo grande (unsigned) ou um número negativo pequeno (signed), dependendo de qual interpretação a instrução aplica.
4. Significa que, ao armazenar um valor com múltiplos bytes na memória, o **byte menos significativo é gravado no endereço mais baixo**, e os bytes mais significativos ficam em endereços progressivamente mais altos.
5. Porque o tipo em C carrega **significado semântico** (o que aquele dado representa: um caractere, um contador, um ponteiro) além do tamanho, e essas decisões de significado são resolvidas pelo compilador. Em Assembly, o que resta é apenas "quantos bytes" e "como esta instrução específica interpreta esses bytes" (signed/unsigned, por exemplo) — não há mais nenhuma noção de "tipo" no sentido de C.
6. `0x2A` = 42 em decimal = `0010 1010` em binário (8 bits).
7. Como **unsigned**: `1000 0000` = 128. Como **signed** (complemento de dois, 8 bits): o bit mais significativo é 1, então é negativo. Invertendo os bits (`0111 1111`) e somando 1 (`1000 0000`)... na verdade o cálculo direto é: valor = -128 (esse é o caso especial onde o bit de sinal sozinho, em complemento de dois de 8 bits, representa exatamente -128).
8. `0xDEADBEEF` em little-endian: endereço `0x2000` = `0xEF` (byte menos significativo), `0x2001` = `0xBE`, `0x2002` = `0xAD`, `0x2003` = `0xDE` (byte mais significativo).
9. `char x` → **byte**; `long y` → **qword**; `int z` → **dword**; `short w` → **word**.

---

*Módulo anterior: [Módulo 3 — Registradores](./03-registradores.md)*
*Próximo módulo: [Módulo 5 — Memória](./05-memoria.md)*
