# Módulo 5 — Memória

> **Arquitetura:** x86-64
> **Sistema operacional de referência:** Linux
> **ABI:** System V AMD64
> **Sintaxe:** Intel

## 1. Introdução

Até aqui, trabalhamos majoritariamente com registradores. Este módulo aborda o outro grande lugar onde dados vivem: a **memória**. O objetivo é que, ao final, você consiga olhar para:

```asm
mov eax, [rbx]
```

e entender, sem hesitar: *"estou lendo um valor que está no endereço armazenado em RBX"* — e depois avançar para formas mais complexas, como `[rbx+8]` e `[rbx+rcx*4]`.

## 2. Endereço vs. valor: a distinção mais importante deste módulo

Esta é a ideia central de todo o módulo, então vamos isolá-la logo no início:

> **Um registrador pode conter um valor comum, ou pode conter um endereço de memória. A CPU não distingue os dois por si só — quem distingue é a instrução.**

Considere:

```asm
mov rax, 5        ; RAX contém o valor 5
mov rbx, 0x1000    ; RBX contém o número 0x1000
```

Nas duas instruções acima, sintaticamente, nada muda — ambas colocam um número em um registrador. A diferença aparece em **como esse número vai ser usado depois**:

```asm
mov ecx, ebx        ; ECX = valor de RBX (o número 0x1000 em si)
mov ecx, [rbx]       ; ECX = o valor ARMAZENADO NO ENDEREÇO 0x1000
```

Repare os colchetes `[ ]`. Eles são o elemento sintático que muda tudo:

- **Sem colchetes**: você está lendo o **valor que está dentro do registrador**.
- **Com colchetes**: você está tratando o valor do registrador **como um endereço**, e pedindo para ler o que está **armazenado naquele endereço** na memória.

## 3. Dereferência: "ir até lá"

O ato de usar um endereço para acessar o valor armazenado nele se chama **dereferência** (*dereference*). Em Assembly, dereferenciar é simplesmente colocar colchetes ao redor do que contém o endereço.

```asm
mov eax, [rbx]
```

Lendo isso em português, passo a passo:

1. `rbx` contém um número — vamos supor que seja `0x2000`.
2. Os colchetes dizem: "não quero o número 0x2000 em si — quero **o que está armazenado no endereço 0x2000**".
3. `mov eax, ...` diz: "pegue esse valor e coloque em EAX".

Resultado: **"Estou lendo um valor que está no endereço armazenado em RBX."** — exatamente a frase-alvo deste módulo.

### 3.1 Uma analogia

Pense em um registrador como um **post-it com um número escrito**. Se o número for `0x2000`, existem duas perguntas possíveis:

- "Qual é o número escrito no post-it?" → resposta: `0x2000` (isso é `mov eax, ebx`, sem colchetes).
- "Vá até a gaveta de número `0x2000` e me diga o que tem dentro dela." → isso é dereferência (`mov eax, [rbx]`, com colchetes).

O post-it (registrador) guarda um **endereço**; a gaveta (memória) guarda um **valor**. São coisas diferentes, e a diferença sintática (colchetes) é o que informa qual das duas você quer.

## 4. Escrita em memória

A mesma lógica se aplica para escrever, não só para ler:

```asm
mov [rbx], eax
```

Isso significa: "pegue o valor que está em EAX e **escreva-o no endereço de memória armazenado em RBX**". Note que a posição dos colchetes indica o **destino** — aqui, o destino é a memória, não um registrador.

Compare os quatro casos possíveis:

```asm
mov eax, ebx      ; registrador ← registrador (copia o valor de EBX para EAX)
mov eax, [ebx]    ; registrador ← memória (lê o que está no endereço em EBX)
mov [ebx], eax    ; memória ← registrador (escreve o valor de EAX no endereço em EBX)
mov eax, 5        ; registrador ← valor imediato (constante)
```

> **Regra prática de leitura:** sempre olhe se há colchetes, e se estão no operando de origem (direita) ou destino (esquerda, em sintaxe Intel). Isso te diz imediatamente se a instrução está lendo memória, escrevendo em memória, ou apenas movendo valores entre registradores.

## 5. Ponteiros: a ideia de C que corresponde a isso

Se você já viu ponteiros em C, a essa altura a conexão deve estar ficando clara:

```c
int  a = 10;
int *p = &a;      // p guarda o ENDEREÇO de a
int  b = *p;      // b recebe o VALOR armazenado no endereço que p guarda (dereferência)
```

A correspondência conceitual:

| C | Assembly (conceito) |
|---|---|
| `p` (o ponteiro em si, um endereço) | um registrador contendo um endereço, ex.: `rbx` |
| `&a` (obter o endereço de uma variável) | calcular um endereço (veremos `lea` no Módulo 6) |
| `*p` (dereferenciar o ponteiro) | usar colchetes: `[rbx]` |

**Importante, reforçando um ponto do Módulo 4:** essa correspondência é conceitual, não uma tradução mecânica. O compilador pode manter `p` em um registrador o tempo todo, ou pode guardá-lo na própria memória (na stack, por exemplo) e carregá-lo em um registrador só quando necessário. Assembly não tem "variáveis" com nome — tudo se resume a registradores e endereços.

## 6. Cada endereço aponta para 1 byte

No fechamento do Módulo 4, ficou a promessa: *"Isso será revisitado no Módulo 5, quando lidarmos com leitura e escrita direta de memória."* É exatamente esse o assunto agora.

Voltando a algo mencionado no Módulo 4: em x86-64, **cada endereço de memória identifica exatamente 1 byte**. Isso significa que endereços consecutivos (`0x1000`, `0x1001`, `0x1002`...) representam bytes vizinhos na memória.

Imagine a memória a partir de RBX = 0x1000 assim:

```
Endereço:   0x1000  0x1001  0x1002  0x1003  0x1004  0x1005 ...
Byte:         78      56      34      12      FF      00
```

Quando você **lê** 1 byte com `mov al, [rbx]`, a CPU olha só para o endereço 0x1000 e traz o valor `0x78`. Nada além disso é tocado:

```
mov al, [rbx]     ; lê 1 byte → AL = 0x78

Endereço:   0x1000  0x1001  0x1002  0x1003
Byte:        [78]     56      34      12
              ↑
             lido
```

Já quando você **lê** 4 bytes com `mov eax, [rbx]` (uma dword), a CPU não lê só o byte de RBX — ela lê RBX e os 3 bytes seguintes (RBX+1, RBX+2, RBX+3), e combina os quatro em little-endian (Módulo 4, Seção 6) para formar um único valor de 32 bits:

```
mov eax, [rbx]    ; lê 4 bytes → EAX = 0x12345678

Endereço:   0x1000  0x1001  0x1002  0x1003
Byte:        [78]    [56]    [34]    [12]
              ↑_______↑_______↑_______↑
                    combinados em little-endian

EAX = 0x12345678
```

Repare que é exatamente o mesmo bloco de memória do exemplo do Módulo 4 (Seção 6) — só que lá o valor estava sendo **escrito**, e aqui está sendo **lido**. A lógica é simétrica: escrever espalha os bytes na ordem little-endian; ler recombina os mesmos bytes de volta.

**A regra geral:**

| Instrução | Registrador de destino | Bytes lidos a partir de RBX |
|---|---|---|
| `mov al, [rbx]` | AL (8 bits) | 1 byte — só `[rbx]` |
| `mov ax, [rbx]` | AX (16 bits) | 2 bytes — `[rbx]` e `[rbx+1]` |
| `mov eax, [rbx]` | EAX (32 bits) | 4 bytes — `[rbx]` até `[rbx+3]` |
| `mov rax, [rbx]` | RAX (64 bits) | 8 bytes — `[rbx]` até `[rbx+7]` |

> O tamanho do registrador de destino não é só "onde o valor vai parar" — ele também dita **quantos bytes a CPU vai buscar na memória** a partir do endereço indicado. É o mesmo princípio do Módulo 4, Seção 3 (tamanho determina interpretação), agora aplicado à leitura de memória em vez de a um valor já em registrador.

## 7. Offsets: acessando além do endereço-base

Agora avançamos para a forma:

```asm
mov eax, [rbx+8]
```

Isso significa: "calcule o endereço somando 8 ao valor de RBX, e leia o que está armazenado **nesse** endereço resultante". Se `rbx` valer `0x2000`, essa instrução lê o valor a partir do endereço `0x2008`.

### 7.1 Por que isso é útil?

Isso é exatamente como structs em C e elementos de arrays são acessados. Considere:

```c
struct Ponto {
    int x;   // offset 0
    int y;   // offset 4
};
```

Se `rbx` guarda o endereço de uma variável `struct Ponto`, então:

```asm
mov eax, [rbx]      ; lê p.x  (offset 0, o int começa logo no endereço-base)
mov eax, [rbx+4]    ; lê p.y  (offset 4, pois x ocupa 4 bytes — um int)
```

O deslocamento (**offset**) de `+4` existe porque `x` (um `int`, 4 bytes) ocupa os primeiros 4 bytes da struct, e `y` começa logo depois.

## 8. Escalonamento: `[rbx+rcx*4]`

A forma mais completa de endereçamento em x86-64 combina uma **base**, um **índice**, um **fator de escala** e um **deslocamento**:

```
[base + índice*escala + deslocamento]
```

Onde a escala só pode ser 1, 2, 4 ou 8 (não qualquer número).

```asm
mov eax, [rbx+rcx*4]
```

Isso significa: "calcule o endereço como (valor de RBX) + (valor de RCX × 4), e leia o que está armazenado nesse endereço".

### 8.1 Por que multiplicar por 4?

Essa forma é o padrão para **acessar elementos de um array**. Considere:

```c
int arr[10];
int valor = arr[i];
```

Se `rbx` guarda o **endereço do início do array** (`arr`), e `rcx` guarda o **índice** `i`, então:

```asm
mov eax, [rbx+rcx*4]
```

lê `arr[i]`. O `*4` existe porque cada `int` ocupa 4 bytes — então, para "pular" `i` elementos a partir do início do array, é preciso avançar `i * 4` bytes, não apenas `i` bytes.

De forma geral: **o fator de escala corresponde ao tamanho de cada elemento do array**. Um array de `char` (1 byte cada) usaria escala 1; um array de `long` ou de ponteiros (8 bytes cada) usaria escala 8.

### 8.2 Juntando tudo: base + índice*escala + deslocamento

A forma mais geral também pode incluir um deslocamento fixo, útil para arrays dentro de structs, por exemplo:

```asm
mov eax, [rbx+rcx*4+12]
```

"Calcule o endereço como RBX + (RCX × 4) + 12, e leia o valor armazenado ali."

Você não precisa memorizar essa fórmula abstratamente — o importante é conseguir, diante de qualquer expressão entre colchetes, identificar cada uma das quatro partes (base, índice, escala, deslocamento) e entender **o cálculo de endereço** que está sendo feito antes mesmo de pensar em qual valor será lido.

## 9. Processo de leitura para expressões de memória

Sempre que encontrar algo como `[expressão]` em Assembly, siga este pequeno processo mental:

1. **Existe base?** (um registrador sozinho, como `rbx`)
2. **Existe índice e escala?** (algo como `rcx*4`)
3. **Existe deslocamento?** (um número somado ou subtraído, como `+8` ou `-4`)
4. **Calcule o endereço final** somando essas partes.
5. **Só depois**, pergunte: "qual valor está armazenado nesse endereço, e de que tamanho?" (o tamanho vem do registrador de destino ou de um sufixo `byte`/`word`/`dword`/`qword`, como vimos no Módulo 4).

Separar "calcular o endereço" de "ler o valor naquele endereço" é a chave para não se confundir com expressões mais longas.

## 10. Exemplo prático integrado

```asm
mov rbx, 0x3000     ; RBX = 0x3000 (um endereço, por hipótese)
mov rcx, 2           ; RCX = 2 (um índice)
mov eax, [rbx+rcx*4] ; EAX = valor no endereço (0x3000 + 2*4) = 0x3008
```

Passo a passo:

1. `mov rbx, 0x3000` — RBX passa a conter o número `0x3000`. Neste ponto, ainda não sabemos se ele será usado como valor comum ou como endereço — isso só fica claro quando aparece entre colchetes.
2. `mov rcx, 2` — RCX passa a conter o valor 2, que será usado como índice.
3. `mov eax, [rbx+rcx*4]` — calcula-se o endereço: `0x3000 + (2 × 4) = 0x3008`. Em seguida, lê-se uma **dword** (4 bytes, porque o destino é `EAX`) a partir do endereço `0x3008`, e o resultado é colocado em `EAX`.

Se pensarmos nisso como um array de `int` (`int arr[]`) começando em `0x3000`, esta instrução equivale a `eax = arr[2]`.

## 11. Exercícios

### Nível 1 — Conceitual

1. Qual é a diferença sintática, em Assembly, entre "usar o valor de um registrador" e "usar o valor de um registrador como endereço"?
2. O que significa "dereferenciar" um endereço?
3. Por que `mov [rbx], eax` escreve na memória, enquanto `mov eax, ebx` não?
4. Por que, em x86-64, dizemos que "cada endereço aponta para 1 byte"?

### Nível 2 — Interpretar instruções isoladas

Para cada instrução abaixo, descreva em português o que ela faz (sem executar valores — apenas a operação):

5. `mov edx, [rax]`
6. `mov [rsi+16], ecx`
7. `mov eax, [rdi+rsi*8]`
8. `mov eax, [rbx+rcx*2+4]`

### Nível 3 — Acompanhar registradores e memória

9. Dado que `rbx = 0x4000`, e a memória contém os bytes `0x4000: 0A`, `0x4001: 0B`, `0x4002: 0C`, `0x4003: 0D`, qual é o valor de `EAX` após `mov eax, [rbx]`? (Lembre-se de little-endian.)

10. Considerando o array conceitual `int arr[4] = {10, 20, 30, 40};` armazenado a partir do endereço `0x5000` (guardado em `rbx`), e `rcx = 3`, o que a instrução `mov eax, [rbx+rcx*4]` coloca em `EAX`?

---

## 12. Respostas

1. A presença de colchetes `[ ]`. Sem colchetes, o valor do registrador é usado diretamente. Com colchetes, o valor do registrador é tratado como um **endereço**, e o que é lido/escrito é o conteúdo **armazenado** naquele endereço.
2. Significa usar um valor que representa um endereço para acessar o dado que está **armazenado** naquele endereço, em vez de usar o endereço em si como valor.
3. Porque `[rbx]` (com colchetes) indica acesso à memória no endereço contido em RBX, enquanto `ebx` (sem colchetes) é apenas o valor bruto armazenado no próprio registrador. `mov [rbx], eax` tem colchetes no destino, então escreve na memória; `mov eax, ebx` não tem colchetes em nenhum operando, então é uma simples cópia entre registradores.
4. Porque essa é a granularidade mínima de endereçamento da arquitetura: cada valor de endereço identifica exatamente um byte. Ler tamanhos maiores (word, dword, qword) significa ler múltiplos bytes a partir de um endereço inicial, combinando-os conforme a ordem little-endian.
5. Lê uma dword (4 bytes, pois o destino é `EDX`) do endereço armazenado em `RAX`, e coloca o resultado em `EDX`.
6. Escreve o valor de `ECX` (uma dword) no endereço calculado como `RSI + 16`.
7. Calcula o endereço como `RDI + (RSI × 8)`, e lê uma dword desse endereço, colocando o resultado em `EAX`. (Escala 8 sugere um array de elementos de 8 bytes, como `long` ou ponteiros.)
8. Calcula o endereço como `RBX + (RCX × 2) + 4`, e lê uma dword desse endereço, colocando o resultado em `EAX`. (Escala 2 sugere um array de elementos de 2 bytes, como `short`.)
9. Os bytes em `0x4000`–`0x4003` são `0A 0B 0C 0D` (do endereço mais baixo ao mais alto). Em little-endian, o byte no endereço mais baixo é o menos significativo. Logo, `EAX = 0x0D0C0B0A`.
10. O array começa em `0x5000`, cada `int` ocupa 4 bytes. `RCX = 3` é o índice. O endereço calculado é `0x5000 + (3 × 4) = 0x500C`, que corresponde a `arr[3]`. Logo, `EAX` recebe o valor `40`.

---

*Módulo anterior: [Módulo 4 — Dados](./04-dados.md)*
*Próximo módulo: [Módulo 6 — Instruções Fundamentais](./06-instrucoes-fundamentais.md)*
